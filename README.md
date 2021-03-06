# Pastry

An open source, pure-[Go](http://www.golang.org "Pretty much the best programming language ever") implementation of the [Pastry Distributed Hash Table](http://en.wikipedia.org/wiki/Pastry_(DHT\) "Pastry on Wikipedia").

## Status
[![Build Status](https://secure.travis-ci.org/secondbit/pastry.png)](http://travis-ci.org/secondbit/pastry)

**Alpha**: Pastry is still in active development. It should not be used for mission-critical software. While it *appears* to function as it should, we're still testing it to make sure there aren't any issues.

## Requirements

This implementation of Pastry is written to be compatible with Go 1. It uses nothing outside of the Go standard library. Nodes in the network must be able to communicate using TCP over a configurable port. Nodes also must be able to have long-running processes.

Pastry was developed on OS X 10.8.1, using Go 1.0.3. It has been verified to work as expected running under Ubuntu 12.04 LTS (64-bit), using Go 1.0.3.

## Installation

The typical `go get secondbit.org/pastry` will install Pastry.

## Documentation

We took pains to try and follow [the guidelines](http://golang.org/doc/articles/godoc_documenting_go_code.html "Godoc guidelines on golang.org") on writing good documentation for `godoc`. You can view the generated documentation on the excellent [GoPkgDoc](http://go.pkgdoc.org/secondbit.org/pastry "Pastry's documentation on GoPkgDoc").

## Use

### Initialising the Cluster

The "Cluster" represents your network of nodes. The first thing you should do in any application that uses Pastry is initialise the cluster.

First, you need to create the local Node&mdash;because Pastry is a peer-to-peer algorithm, there's no such thing as a server or client; instead, everything is a "Node", and only Nodes can connect to the Cluster.

```go
hostname, err := os.Hostname()
if err != nil {
	panic(err.Error())
}
id, err := pastry.NodeIDFromBytes([]byte(hostname+" test server"))
if err != nil {
	panic(err.Error())
}
node := pastry.NewNode(id, "your_local_ip_address", "your_global_ip_address", "your_region", 8080)
```

NewNode expects five parameters:

1. The ID of the new Node. We created one in the code sample above. The ID can be any unique string&mdash;it is used to identify the Node to the network. The ID string has to be over 16 bytes long to be substantial enough to form an ID out of, or NodeIDFromBytes will return an error.
2. Your local IP address. This IP address only needs to be accessible to your Region (a concept that will be explained below).
3. Your global IP address. This IP address should be accessible to any Node in your network&mdash;the entire Internet should be able to reach the IP.
4. Your Region. Your Region is a string that helps segment your Pastry network to keep bandwidth minimal. For cloud providers (e.g., EC2), network traffic within a region is free. To take advantage of this, we modified the Pastry algorithm to use the local IP address when two Nodes are in the same Region, and the global IP address the rest of the time, while heavily favouring Nodes that are in the same Region. This allows you to have Nodes in multiple Regions in the same Cluster while minimising your bandwidth costs.
5. The port this Node should listen on, as an int. Should be an open port you have permission to listen on.

Once you have a Node, you can join the Cluster.

```go
cluster := pastry.NewCluster(node, credentials)
```

NewCluster just creates a Cluster object, initialises the state tables and channels used to keep the algorithm concurrency-safe, and returns it. It requires that you specify the current Node and supply [Credentials](http://go.pkgdoc.org/secondbit.org/pastry#Credentials) for the Cluster.

Credentials are an interface that Pastry defines to help control access to your clusters. Credentials could be whatever you want them to be: public/private keys, a single word or phrase, a rather large number... anything at all is fair game. The only rules for Credentials are as follows:

1. Calling `Marshal()` on any implementation of Credentials must return a slice of bytes.
2. Calling `Valid([]byte)` on any implementation of Credentials must decide whether the specified slice of bytes should grant access to the Cluster (return true) or not (return false). The recommended way to do that is to attempt to unmarshal the byte slice into your Credentials implementation (returning false on error) and then comparing the resulting instance with your local instance. But there's nothing stopping you from just returning true, granting anyone who cares to connect full access to your Cluster. Like [PSN](http://en.wikipedia.org/wiki/PlayStation_Network_outage) does (*Zing!*)

In the event that `Valid([]byte)` returns false for *any reason*, the Node will not be added to the state tables of the current Node. It will not be notified that its attempt failed, but it will not receive any messages from the Cluster.

### Listening For Messages

To participate in the Cluster, you need to listen for messages. You'll either be used to pass messages along to the correct Node, or will receive messages intended for your Node.

```go
cluster.Listen()
defer cluster.Stop()
```

`Listen()` is a blocking call, so if you need it to be asynchronous, throw it in a goroutine. **Note**: If you listen twice on the same Cluster in two different goroutines, concurrency-safety **is compromised**. You should only ever have one goroutine Listen to any given Cluster.

`Stop()` ends the Listen call on a Cluster. You'll not receive messages, and will stop participating in the Cluster. It is the graceful way for a Node to exit the Cluster.

### Registering Handlers For Your Application

Pastry offers several callbacks at various points in the process of exchanging messages within your Cluster. You can use these callbacks to register listeners within your application. These callbacks are simply instances of a type that fulfills the [pastry.Application](http://go.pkgdoc.org/secondbit.org/pastry#Application) interface and are subsequently registered to a cluster.

```go
type PastryApplication struct {
}

func (app *PastryApplication) OnError(err error) {
	panic(err.Error())
}

func (app *PastryApplication) OnDeliver(msg pastry.Message) {
	fmt.Println("Received message: ", msg)
}

func (app *PastryApplication) OnForward(msg *pastry.Message, next pastry.NodeID) bool {
	fmt.Printf("Forwarding message %s to Node %s.", msg.ID, next)
	return true // return false if you don't want the message forwarded
}

func (app *PastryApplication) OnNewLeaves(leaves []*pastry.Node) {
	fmt.Println("Leaf set changed: ", leaves)
}

func (app *PastryApplication) OnNodeJoin(node *pastry.Node) {
	fmt.Println("Node joined: ", node.ID)
}

func (app *PastryApplication) OnNodeExit(node *pastry.Node) {
	fmt.Println("Node left: ", node.ID)
}

func (app *PastryApplication) OnHeartbeat(node *pastry.Node) {
	fmt.Println("Received heartbeat from ", node.ID)
}

app := &PastryApplication{}
cluster.RegisterCallback(app)
```

The methods will be invoked at the appropriate points in the lifecycle of the cluster. You should consult [the documentation](http://go.pkgdoc.org/secondbit.org/pastry#Application) for more information.

### Announcing Your Presence

Finally, to join a Cluster that has already been formed (which you'll want to do, unless this is the first server in the group you're standing up), you're going to need to use the `Join` method to announce your presence and initialise your state tables. The `Join` method is simple:

```go
cluster.Join("127.0.0.1", 8080)
```

The first parameter is simply the IP address of a known Node in the Cluster, as a string. The second is just the port, as an int.

When `Join()` is called, the Node will contact the specified Node and announce its presence. The specified Node will send the joining Node its state tables and route the join message to the other Nodes in the Cluster, who will also send the joining Node their state tables. These state tables will initialise the joining Node's state tables, allowing it to participate in the Cluster.

### Sending Messages

Sending a message in Pastry is a little weird. Each message has an ID associated with it, which you can generate based on the contents of the message or some other key. Pastry doesn't care what the relationship between the message and the ID is (Pastry is perfectly happy with random message IDs, in fact), but applications built on Pastry sometimes dictate the terms of the message ID. All Pastry requires is that your message ID, like your Node IDs, has at least 16 bytes worth of data in it.

Messages in Pastry aren't sent *to* something, they're sent *towards* something--their message ID. When a Node receives a Message, it checks to see if it knows about any Node with a NodeID closer to the MessageID than its own NodeID. If it does, it forwards the Message on to that Node. If it doesn't it considers the Message to be delivered. There are all sorts of algorithms in place to help the Message reach that delivery quicker, but they're not really the important bit. The important bit is that messages aren't sent *to* Nodes, they're sent *towards* their MessageID.

Here's an example of routing a Message with a randomly generated ID (based on the `crypto/rand` package) through a Cluster:

```go
b := make([]byte, 16)
_, err := rand.Read(b)
if err != nil {
	panic(err.Error())
}
id, err := pastry.NodeIDFromBytes(b)
if err != nil {
	panic(err.Error())
}
purpose := byte(16)
msg := cluster.NewMessage(purpose, id, []byte("This is the body of the message."))
err = c.Send(msg)
if err != nil {
	panic(err.Error())
}
```

You'll notice we set `purpose` in there to `byte(16)`. Purpose is a way of distinguishing between different types of Messages, and is useful when handling them. We only guarantee that bytes with values 16 and above will go unused by Pastry's own messages. To avoid collisions, you should only use bytes with values of 16 and above when defining your messages.

We repeated that because it's kind of important.

## Contributing

We'd love to see Pastry improve. There's a lot that can still be done with it, and we'd love some help figuring out how to automate some more complete tests for it.

To contribute to Pastry:

* **Fork** the repository
* **Modify** your fork
* **Send** a pull request
	* Bonus points if the pull request includes *what* you changed, *why* you changed it, and *has unit tests* attached.
	* For the love of all that is holy, please use `go fmt` *before* you send the pull request.

We'll review it and merge it in if it's appropriate.

## Implementation Details

We approached this pragmatically, so there are some differences between the Pastry specification (as we understand it) and our implementation. The end result should not be materially changed.

* We *removed the concept of a Neighborhood Set*. The Neighborhood Set was intended to keep information on the proximity of Nodes. We opted to instead store the proximity information inside each Node object.
* We *store every Node in the Routing Table*. The specification dictates that when two Nodes compete for the same space in the Routing Table, only the closest (based on proximity) is stored. We opted to store both, then *route* based on proximity (when routing a message, we select the Node with the closest proximity score). In clusters of sufficiently large size (thousands), this may create memory concerns, as we estimate that Nodes may occupy roughly a couple KB in memory. For our purposes, the memory cost isn't a concern, and it greatly simplifies the algorithm.
* We introduced the concept of Regions. Regions are used to partition your Cluster and give preference to Nodes that are within the same Region. It is useful on cloud providers like EC2 to minimise traffic between regions, which tends to cost more than traffic on the local network. This is implemented as a raw multiplier on the proximity score of nodes, based on if the regions match or not. It should not materially affect the algorithm, outside the intended bias towards local traffic over global traffic.

## Known Bugs

* If you should happen to have two Nodes in a Cluster who don't agree as to what time it is, it's possible to get them stuck in an infinite loop that saturates the network with messages. For the love of God, use NTP to make your Nodes agree what time it is. (*Note*: This is to prevent race conditions when two Nodes join simultaneously.)
* In the event that: 1) a Node is added, 2) the Node receives a message *before* it has finished initialising its state tables, and 3) the Node, based on its partial implementation of the state tables, is the closest Node to the message ID, that Node will incorrectly assume it is the destination for the message when there *may* be a better suited Node in the network. Depending on network speeds and the size of the cluster, this period of potential-for-message-swallowing is expected to last, at most, a few seconds, and will only occur when a Node is added to the cluster. If, as per the previous bug, your Nodes don't agree on the time… well, God help you.
* In the event that one of the two immediate neighbours (in the NodeID space) of the current Node leaves the cluster, the Node will have a hole in its leaf set until it next receives (or has a reason to request) state information from another Node. This should not affect the outcome of the routing process, but may lead to sub-optimal routing times.

## Authors

The following people contributed code that found its way into Pastry:

* Paddy Foran ([paddy@secondbit.org](mailto:paddy@secondbit.org))

## Contributors

The following people contributed to the creation of Pastry through advice and support, not through code:

* [Matthew Turland](http://www.matthewturland.com) offered support and advice, and has been invaluable in bringing the software to fruition.
* [Chris Hartjes](http://www.littlehart.net/atthekeyboard) offered feedback and advice on our testing strategies.
* [Jesse McNelis](http://jessta.id.au) provided his services both as a bug-hunter and as a rubber duck.
* [Dr. Steven Ko](http://www.cse.buffalo.edu/people/?u=stevko) of the University at Buffalo offered valuable feedback on Pastry and Distributed Hash Tables in general.
* [Jan Newmarch's excellent guide to writing networking code in Go](http://jan.newmarch.name/go/) gave us valuable information.
* [The Go Community](https://groups.google.com/group/go-nuts) (which is superb), offered advice and feedback throughout the creation of this software.

## License

Copyright (c) 2012 Second Bit LLC

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
