# Iteration 3 - Networking Basics

The previous iterations have been focused on developing the core
data structures and algorithms of our currency system. We're now going
to take a bit of a detour and dive into another very important aspect
of the project -- networking.

One of Bitcoin's fundamental design principles is decentralization.
When we talk about "Bitcoin" as a whole, we're really talking about the emergent behavior of a vast,
worldwide network of computers all running bitcoin client software
and working independently to guide the development of the block chain
and its related data.

The ability of the network to function in this way is one of the most
fundamental strengths of a system like Bitcoin -- since the network
as a whole determines what happens with regard to the block chain,
processing transactions, etc, we don't have to rely on trust in one
single entity to ensure that our system functions properly. However
it also imposes some heavy constraints on the system, as the protocol
and core processes need to be carefully designed to protect the
network from getting into trouble (or from being exploited by attackers).

In this section, we'll learn about some of these network principles
and dig into the nitty gritty of our communication protocol.

## P2P Basics

The first point to emphasize about our networking approach is that it
requires full **peer-to-peer** communication. This may be different from
other network programming you have done in the past, where an asymmetrical,
client/server model tends to be more common.

In a traditional client/server model, the server responds to requests from the
client by sending relevant information, but does not *initiate* requests of its own.
Peer-to-Peer (P2P) networks, by contrast, emphasize symmetrical, bi-directional
communication between nodes. Node A might send a message to Node B and receive
a response, while a few moments later Node B might send a completely unrelated
message to Node A and get a response of its own.

To make another analogy to a traditional client/server model, this means that
in a P2P network each node behaves as both a server *and* a client. As a server
the node will receive messages from other nodes and send appropriate responses,
but as a client it will initiate its own requests as needed.

## Transit Mechanism and Formats (TCP / JSON)

So how will all this work from a technical perspective? For our network, we'll
be using [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) as a
*transit protocol* to send data over the network between nodes. Each node will
need to run a simple TCP Server in order to field requests from other nodes.
Additionally, for every *peer* to which a node wants to connect, they will need
to create a TCP Client that connects to that node at the appropriate IP Address
and Port.

As for the actual messages themselves, we'll be using JSON as a serialization
format. In fact, all messages will follow a simple JSON format including a
`message_type` that identifies the kind of message being sent and an optional `payload`
that includes any necessary information.

So an example message might look like:

```json
{
  "message_type": "add_peer",
  "payload": "10.0.1.2:3000"
}
```

__Why these technologies?__

There are a variety of transit protocols and serialization formats out there that
we could choose from. We choose TCP because it has great built-in procedures for
ensuring reliability of the messages -- for example the protocol is able to intelligently
retry certain packets if they fail due to network issues, etc. Additionally, TCP defines
robust procedures for splitting arbitrary messages across packets and guaranteeing that
these packets will arrive in the proper order to avoid garbling our data.
Finally TCP is a very common and well-supported protocol, and you'll find reliable tools
for working with it in any major programming language.

As for JSON, it's an extremely popular serialization format these days, and has the
additional advantage of being extremely human-readable. If we were more concerned with
efficiency and performance (both from a speed and bandwidth perspective), we might want
to investigate a dedicated binary protocol (as the actual Bitcoin protocol uses), but
for our purposes we are more interested in clarity and ease of use.

## Message Types

## Automated TCP Protocol Spec

##
