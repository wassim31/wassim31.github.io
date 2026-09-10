---
layout: post
title: "From BitTorrent to IPFS: Understanding the Network Behind Decentralized Files"
date: 2026-09-10 10:00:00 +0000
description: "How the limits of client-server storage lead to peer-to-peer networks, why finding data without a central directory needs a DHT, and how Kademlia's XOR distance gets IPFS to the right peer."
tags: [networking, distributedsystems, p2p, ipfs]
---

Think about the simplest way to share a file over a network. You put it on a server, and whoever wants it asks that server for it.

```
              File Server
             /     |     \
            /      |      \
        Client A Client B Client C
```

Client A asks for `hello.txt`, the server finds it, sends it back. Client B and Client C can ask for the same file, or something else, and the server just keeps answering. That's the whole model. One machine owns the data, one machine is responsible for handing it out, and everyone else just needs to know its address.

This works, and for a lot of real systems it's genuinely the right answer. It's simple to reason about. If you want to know whether a file exists, you ask the server. If you want to know who has permission to read it, you ask the server. There's exactly one place that has to be right.

The trouble starts when the number of clients grows, or the amount of data grows, or both. A single server has finite bandwidth, finite disk, finite everything. At some point it becomes the bottleneck for every single request in the system, even requests that have nothing to do with each other. And there's a sharper problem underneath that: if that one machine goes down, or the network link to it dies, every client is stuck, no matter how healthy they are individually. The server isn't just doing the work, it's the single point that everything depends on.

There's also a quieter cost that people notice later than the outage: locality. If your server sits in one datacenter and your clients are scattered across the world, everyone pays the round trip to that datacenter, every time, for every file, even files that a hundred other people in their own city already have a copy of.

None of this means client-server is a bad design. It means it's a design with a specific shape of tradeoff, and once you outgrow it, you start looking at ways to spread that responsibility across more than one machine.

"Distributed" gets used pretty loosely though, so let's be precise about it. A traditional client-server filesystem, something like NFS, still has a clearly defined server. Clients mount it, and every read and write goes to that server. What people usually mean by a distributed filesystem, something like HDFS or Ceph, is a system that spreads the actual storage, and often the metadata too, across many machines. But even there you'll frequently find at least one component that's still centrally responsible for something: HDFS has historically had a NameNode that tracks where every block lives, which is its own kind of single point even if the data itself is spread out. Distributing storage and distributing responsibility for coordinating that storage are two different problems, and a lot of systems only solve the first one.

## From client-server to peer-to-peer

So here's a genuinely different question: what if the machines holding the data didn't just sit there waiting to be asked, but actually participated in helping other machines find and retrieve it too?

Instead of this:

```
                  Server
                /   |   \
               /    |    \
             C1     C2    C3
```

picture this instead:

```
        A <------> B
        ↕  \      / ↕
        ↕    \  /   ↕
        C <--> D <--> E
```

No single machine here is required to know about every file in the system. Every node can ask for data, and every node can be asked for data. This is the basic idea behind peer to peer networks, and if you've ever run BitTorrent, you've already lived inside this model without necessarily thinking about it in these terms. There's no BitTorrent server holding the movie you're downloading. There's a swarm of other people's machines, each holding pieces, each willing to send you a piece and receive one from you at the same time.

Getting rid of the central server solves the single-point-of-failure problem pretty directly. If B goes offline, A can still reach D, and D can still reach C and E. Nobody's request depends on one specific machine being alive.

But it also creates a problem the client-server model never had to deal with, because the client-server model solved it for free just by existing: if there's no server, and no central directory, how does a peer even know where to start looking for a file? In the server model, "where is it" was never a real question, the answer was always "the server." Take the server away, and that question suddenly needs an actual answer.

## The problem of finding data

The simplest way to state what we need is a mapping from a name to whoever has it.

```
key → peer
```

For example:

```
"hello.txt" → Peer X
```

If you had one trusted machine willing to keep that whole mapping for every file in the system, you'd just be rebuilding a central server, this time as a lookup service instead of a storage service. That doesn't actually get you anything, it just moves the same single point of failure one layer up.

The idea behind a distributed hash table, a DHT, is to keep that same kind of mapping, key to peer, but spread the responsibility for holding pieces of it across the peers themselves. No one node holds the whole table. Each node holds a slice of it, and knows how to point you toward whoever holds the rest.

That's the shape of the problem. It doesn't yet tell you how a node decides which slice it's responsible for, or how you find the right node to ask when you don't know who holds what. That's a real algorithm, and the one that IPFS actually uses is called Kademlia.

## How Kademlia finds its way

Kademlia is one specific way to build a DHT. It's not the only one, but it's the one worth understanding here, because it's what IPFS runs on underneath.

The first thing Kademlia does is give every node an ID. In a real deployment these are typically 160-bit numbers, but that's painful to reason about on paper, so let's use a toy 4-bit space instead and keep the actual mechanics identical.

```
A = 0000
B = 0011
C = 0101
D = 1001
E = 1010
```

Worth pausing on one thing before going further: a 160-bit ID space has 2^160 possible identifiers. That is an astronomically large number, and it does not mean the network has anywhere close to that many nodes in it. It just means IDs are assigned from a space big enough that two random nodes are essentially never going to collide, the way UUIDs work.

Data gets an ID too, using the same kind of hash. Say we hash the name `hello.txt` and, in our toy 4-bit world, it comes out to:

```
"hello.txt"
     ↓
   hash
     ↓
   1011
```

Now both nodes and data live in the same ID space, which is the whole trick. Finding "who is responsible for hello.txt" becomes finding "which node's ID is closest to 1011."

So, closest by what definition? Kademlia doesn't use geographic distance or ping time. It uses XOR.

```
A = 0000
B = 0011

0000
XOR 0011
---------
0011
```

XOR two IDs together and you get a number that represents how different they are, bit by bit. Do it against our target, `1011`:

```
1010 XOR 1011 = 0001 = 1
1001 XOR 1011 = 0010 = 2
0101 XOR 1011 = 1110 = 14
```

E (`1010`) comes out with a distance of 1, D (`1001`) with a distance of 2, C (`0101`) with a distance of 14. Smaller XOR distance means closer in Kademlia's identifier space. That's it, that's the entire definition of "close" that the whole protocol is built on. It has nothing to do with physical network distance, nothing to do with IP addresses, nothing to do with latency. Two nodes can be XOR-close and sit on opposite sides of the planet, or XOR-far and sit in the same server rack. It's a purely mathematical closeness inside the ID space, not a physical one.

Every node keeps a routing table of other nodes it knows about, organized into buckets, and which bucket a contact lands in depends on how XOR-close it is to you. Here's where it's easy to get the wrong intuition: the bucket isn't determined by how many bits differ. It's determined by the position of the highest bit that differs.

```
A = 0000
B = 0011

A XOR B = 0011
             ^
        highest set bit
```

`0011` has its highest set bit at the second position from the right. That position is what decides the bucket, not the fact that there happen to be two 1s in the result. You can see the distinction clearly by comparing `0011` and `0111`. Different number of 1 bits, three versus two, but the highest set bit sits in the same position in both, so they land in the same bucket.

```
0001 → bucket 1
0010 → bucket 2
0011 → bucket 2
0100 → bucket 3
0101 → bucket 3
0110 → bucket 3
0111 → bucket 3
1000 → bucket 4
```

This is where people who've spent time thinking about byte order in systems code tend to get tripped up, so let's be explicit: "highest" here means most significant in the identifier itself, reading it the normal way, left to right, like a number. It has nothing to do with big-endian or little-endian memory layout. This is a property of the ID as a value, not of how it happens to be stored in RAM.

Scale this up to a real 160-bit identifier and you get up to 160 buckets per node, one for each possible position of the highest differing bit.

So what actually lives inside a bucket? Not just an ID. A bucket holds contacts, and a contact is enough information to actually reach that peer:

```
Bucket:
    Peer B → ID + IP + port
    Peer C → ID + IP + port
    Peer D → ID + IP + port
```

Each bucket can hold up to some fixed number of contacts, usually called `k`. Real Kademlia deployments commonly use `k = 20`. You don't need to memorize the number, the point is just that a bucket has a cap, it's not an unbounded list of every peer that ever happened to be that XOR-distance away.

Which raises a fair question: who actually puts peers into these buckets in the first place? Nobody's administering this by hand.

```
New node A
    |
    ↓
Bootstrap peer
    |
    ↓
Other known peers
    |
    ↓
Routing table
```

A brand new node starts out knowing about one or a handful of bootstrap peers, addresses it was configured with or discovered some other way. From there, every interaction it has with the network, every lookup it does or answers, teaches it about more peers, and it slots them into the right bucket based on their XOR distance from itself. There's no central directory handing out routing tables. Every node builds and maintains its own, purely from the conversations it happens to have.

Let's actually walk through a lookup, because the mechanics only really click once you see the whole thing move.

Say the target is `1011`, and we have:

```
Target = 1011

A = 0000
B = 1001
C = 1010
D = 1111
```

A wants to find whoever is closest to that target. It starts by checking the distance to everyone it already knows:

```
A → target:
0000 XOR 1011 = 1011 = 11

B → target:
1001 XOR 1011 = 0010 = 2

C → target:
1010 XOR 1011 = 0001 = 1

D → target:
1111 XOR 1011 = 0100 = 4
```

C is the closest one A already knows about, distance 1. So A asks C directly. In real Kademlia this is a message called `FIND_NODE`:

```
A --------------> C
    FIND_NODE(1011)
```

What A is actually saying here is simple: "C, tell me about the peers you know that are closest to `1011`." A isn't asking C for the file. A is asking C for better directions.

Now say C's own routing table has peers A has never heard of:

```
E = 1011
F = 1100
G = 1000
H = 0110
```

C checks the distance from each of these to the target:

```
E = 1011
1011 XOR 1011 = 0000 → 0

G = 1000
1000 XOR 1011 = 0011 → 3

F = 1100
1100 XOR 1011 = 0111 → 7

H = 0110
0110 XOR 1011 = 1101 → 13
```

E comes out at distance 0, which means E's ID is literally the target. G is next at 3, F at 7, H trails far behind at 13. C sends A its closest few, E, G, and F, and leaves H out since it's clearly not useful here.

This is the part worth sitting with for a second: A did not know E existed. C did. By asking C, a node that was closer to the answer than A was, A got handed a peer that's closer still. That's the entire engine of the protocol. You don't need a central index of everyone, you just need each hop to know someone who's a little closer than you are.

A now has E in its shortlist, and since E is distance 0, that's as close as it gets. The lookup effectively finishes here:

```
A → C → E
```

Though depending on exactly who knows who, it could just as easily have taken one more hop, something like `A → C → G → E`. Real Kademlia doesn't do this one query at a time either. It keeps a running shortlist of the closest peers found so far and queries several of them in parallel, controlled by a parameter usually called `alpha`, so it converges faster than the strictly serial version I just walked through. The serial version is exactly what's happening conceptually, just spread across a few simultaneous requests instead of one at a time.

Now, an important thing to be clear on before moving to IPFS. Being close to the target ID does not automatically mean that peer is storing the actual data. All Kademlia gives you is a mechanism for locating whoever is responsible, by ID, for a given key. What that responsibility actually means, whether it's "store the full file," or "store a pointer to who has the file," or something else entirely, is up to whatever's built on top of the DHT. Kademlia hands you the address book. It doesn't decide what's written at that address.

## Where IPFS fits

This is exactly the layer IPFS is built on top of, and it matters a lot exactly what IPFS uses the DHT for, and what it doesn't.

IPFS identifies content by what it is, not by where it's stored. Take a file, hash it, and that hash becomes its identifier, called a CID.

```
file
 ↓
hash
 ↓
CID
```

There's no server path here, no `/files/hello.txt`. The identifier is derived entirely from the content itself. Change one byte of the file and you get a completely different CID.

Once you have a CID, the question becomes the same one we already spent this whole article answering: who has it? This is where the DHT, the same Kademlia mechanism from above, comes back in.

```
CID
 ↓
content routing
 ↓
DHT lookup
 ↓
provider peers
 ↓
retrieve content
```

Worth being explicit here, since it's easy to blur: the DHT is used purely to discover which peers can provide a given CID. It doesn't carry the file itself. It's a directory, not a delivery truck.

```
             DHT
              |
       "Who provides CID X?"
              |
       ┌──────┴──────┐
       ↓             ↓
    Peer B         Peer C
       \             /
        \           /
         ↓         ↓
             Data
```

Once the lookup returns a handful of peers who announced that they have the content for that CID, the actual bytes get pulled from those peers directly, over a separate connection, the same peer to peer exchange we talked about at the start.

There's one more wrinkle here: large content in IPFS usually isn't one single blob behind one CID. It gets split into blocks, and each block gets its own CID.

```
movie.mp4
   |
   ├── Block A → CID A
   ├── Block B → CID B
   ├── Block C → CID C
   └── Block D → CID D
```

Which means retrieving one large file can mean discovering providers for several different blocks, possibly different peers for different blocks, and pulling them all in. There's a whole protocol, Bitswap, dedicated to negotiating exactly which blocks get exchanged with which peers, but that's a rabbit hole for another post. The part that matters here is just that content routing and content transfer are two separate concerns, and the DHT only handles the first one.

If BitTorrent proved that a swarm of ordinary machines exchanging pieces directly could actually move real amounts of data at scale, IPFS is answering a related but different question: how do you find the right swarm for a piece of content in the first place, when the content itself, not some server's file path, is the only thing you started with. Kademlia is the answer to that question, borrowed almost directly from the BitTorrent world, where a Kademlia based DHT has quietly been running trackerless swarms for years.

Put the whole chain together and it reads as one continuous story:

```
Client-server
      ↓
Centralized lookup/storage
      ↓
Need to distribute responsibility
      ↓
Peer-to-peer
      ↓
No central directory
      ↓
DHT
      ↓
Kademlia
      ↓
Node IDs + XOR distance + routing buckets
      ↓
Find relevant peers
      ↓
IPFS content routing
      ↓
Find content providers
      ↓
Retrieve the actual content
```

The CID tells us what content we want.

Kademlia gives us a way to navigate the distributed network.

The DHT helps us discover relevant peers.

The peer-to-peer data exchange gets the actual content.
