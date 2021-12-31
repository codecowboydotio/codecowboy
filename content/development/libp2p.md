+++
title = "Libp2p experiments"
date = "2021-12-30"
aliases = ["libp2p"]
tags = ["libp2p", "control plane"]
categories = ["software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro
I recently picked up libp2p again. It's a project that I've been checking in on for a number of years, and decided to check it again recently. The difference is that this time I had a use for it!

You can find out more about libp2p [here](https://libp2p.io)

## What is it?
Libp2p is a peer to peer library for a multitude of languages.
It provides a modular networking stack that you can use in your projects. 

In the words of the site itself: 

**Run your network applications free from runtime and address services, independently of their location.**

Libp2p also has the following high level features:

- Use different transports: Includes WebRTC, QUIC, TCP and more
- Native Roaming: Your application can be moved between machines without configuration.
- Runtime Freedom: Connectivity is abstracted from the platform you're using.
- Protocol Muxing: Reuse existing protocol connections.
- Work Offline: No need for a centralised backbone!
- Encrypted Connections: Protect data in transit but also validate peer identities.
- Easy Upgrades: Baked in versioning of services allows backwards compatiability support.
- Browser Support: Write your app in the browser, but make it work across platforms and networks!
- Latency Independant: Make sure your app is using the best transport for the environment. Handle high latency scenarios.

## In a nutshell
Libp2p allows you to write distributed applications that can run almost anywhere and are independant of the platform they run on.

## How can it be used?
Libp2p can be used to write distributed applications, as I mentioned above. In this case I'm going to demonstrate by writing a distributed control plane component for NGINX Unit.

I wanted to explore the idea of using a peer to peer control plane to synchronise configurations among disparate NGINX unit instances. This made sense to me because unit has an API driven interface to configure it. Rather than using a "control plane" instance, I had thought to myself that every unit instance should have the following qualities.

- Be self contained
- Discover other instances
- Have a way of triggering a configuration notification to other nodes
- Have a REST endpoint.


## Components 
My control plane will have a few conponents. 

- Peer identity
- Secure Connection
- Peer discovery
- Message bus
- REST endpoint

A component diagram is below - I find this helps me visualise how things work. The thing to remember is that everything is peer to peer - which means that there is no central controller for the message bus component. 

![libp2p_component_diagram.JPG](/images/libp2p_component_diagram.JPG)

## Peer Identity
To begin with, we need to set up libp2p in order to make all of this work. While libp2p supports multiple languages, I am going to use javascript in my examples. 

The first thing to do is the import all of the required libraries - you'll also need to *npm install* them as well. 
Note that I'm using *express*, *request* and *body-parser* - each of these is used as part of my REST endpoint later on.

```javascript
const Libp2p = require('libp2p')
const TCP = require('libp2p-tcp')
const { NOISE } = require('libp2p-noise')
const MPLEX = require('libp2p-mplex')
const process = require('process')
const { multiaddr } = require('multiaddr')
const Gossipsub = require('libp2p-gossipsub')
const Bootstrap = require('libp2p-bootstrap')
const bootstrapers = require('./bootstrapers')
const MulticastDNS = require('libp2p-mdns')
const { fromString: uint8ArrayFromString } = require('uint8arrays/from-string')
const { toString: uint8ArrayToString } = require('uint8arrays/to-string')
const express = require('express')
const bodyParser = require('body-parser')
const request = require('request')
```


To begin with, we need to create a libp2p node. The node in its most basic form is an address to listen on, a transport, a muxer and a connection encryption module.

I am also using peer discovery and a pub sub message bus, which I'll cover in detail later.

An interesting note is that I am using TCP for reliability, but am listening on 0.0.0.0 or all interfaces. This has an odd side effect of listening on loopback as well as any other interfaces that present on the server where my code is running.

The basic code looks like this.

```javascript
;(async () => {
  const node = await Libp2p.create({
    addresses: {
      listen: ['/ip4/0.0.0.0/tcp/0']
    },
    modules: {
      transport: [TCP],
      streamMuxer: [MPLEX],
      connEncryption: [NOISE],
      peerDiscovery: [MulticastDNS],
      pubsub: Gossipsub
    },
    config: {
      peerDiscovery: {
        mdns: {
          interval: 60e3,
          enabled: true
        }
      },
      pubsub: {
        enabled: true,
        emitSelf: false
      }
    }
  })

```

## Secure Connection

## Peer Discovery
In order to perform peer discovery, I am using multicast DNS. 
This means that this code is only appropriate for a LAN environment. Fortunately libp2p has a router component for crossing layer three networks, but that's going to become a second post.

I've placed some logging into my code to log out to the console, the peer address that is assigned, as well as when other nodes are discovered. 

Each node that starts will log out its own node ID, as well as the node ID's of any other nodes within the network that are discovered.

```javascript
  node.connectionManager.on('peer:connect', (connection) => {
    console.log('Connection established to:', connection.remotePeer.toB58String())      // Emitted when a peer has been found
  })

  node.on('peer:discovery', (peerId) => {
    // No need to dial, autoDial is on
    console.log('Discovered:', peerId.toB58String())
  })

  console.log('My Node ID: ', node.peerId.toB58String())
//  console.log(node)

  await node.start()
```

## The full code to this point

The full code to this point is as follows:

```javascript
const Libp2p = require('libp2p')
const TCP = require('libp2p-tcp')
const { NOISE } = require('libp2p-noise')
const MPLEX = require('libp2p-mplex')
const process = require('process')
const { multiaddr } = require('multiaddr')
const Gossipsub = require('libp2p-gossipsub')
const Bootstrap = require('libp2p-bootstrap')
const bootstrapers = require('./bootstrapers')
const MulticastDNS = require('libp2p-mdns')
const { fromString: uint8ArrayFromString } = require('uint8arrays/from-string')
const { toString: uint8ArrayToString } = require('uint8arrays/to-string')
const express = require('express')
const bodyParser = require('body-parser')
const request = require('request')

;(async () => {
  const node = await Libp2p.create({
    addresses: {
      listen: ['/ip4/0.0.0.0/tcp/0']
    },
    modules: {
      transport: [TCP],
      streamMuxer: [MPLEX],
      connEncryption: [NOISE],
      peerDiscovery: [MulticastDNS],
      pubsub: Gossipsub
    },
    config: {
      peerDiscovery: {
        mdns: {
          interval: 60e3,
          enabled: true
        }
      },
      pubsub: {
        enabled: true,
        emitSelf: false
      }
    }
  })

  node.connectionManager.on('peer:connect', (connection) => {
    console.log('Connection established to:', connection.remotePeer.toB58String())      // Emitted when a peer has been found
  })

  node.on('peer:discovery', (peerId) => {
    // No need to dial, autoDial is on
    console.log('Discovered:', peerId.toB58String())
  })

  console.log('My Node ID: ', node.peerId.toB58String())
//  console.log(node)

  await node.start()


})();
```

If you copy and paste the code, and have all of the modules installed, it will look something like this:
The first node fires up and is assigned a node ID.
It is constantly listening for other nodes on the network to connect to.

```Bash
[root@node-1 src]# node blog.js
My Node ID:  QmZbKnrjE8MbZzyu1r7cRgzdEL1cpENHrRJzoYAayEPWW3
```

When the second node starts, it likewise assigns a node ID, but discovers the first node.
We can see that the second node discovers the first node by looking at the node ID's.
The last thing that the second node does is create a connection through to the first node.

```Bash
[root@node-2 src]# node blog.js
My Node ID:  QmRbXUDxrme7hRHJX4haYVnHcSWEcfgiD5QLuH1HyUfUSs
Discovered: QmZbKnrjE8MbZzyu1r7cRgzdEL1cpENHrRJzoYAayEPWW3
Connection established to: QmZbKnrjE8MbZzyu1r7cRgzdEL1cpENHrRJzoYAayEPWW3
Connection established to: QmZbKnrjE8MbZzyu1r7cRgzdEL1cpENHrRJzoYAayEPWW3
Connection established to: QmZbKnrjE8MbZzyu1r7cRgzdEL1cpENHrRJzoYAayEPWW3
```

When we come back to node one, it has automatically discovered the other nodes, and has established a connection to them.

```Bash
[root@node-1 src]# node blog.js
My Node ID:  QmZbKnrjE8MbZzyu1r7cRgzdEL1cpENHrRJzoYAayEPWW3
Discovered: QmRbXUDxrme7hRHJX4haYVnHcSWEcfgiD5QLuH1HyUfUSs
Connection established to: QmRbXUDxrme7hRHJX4haYVnHcSWEcfgiD5QLuH1HyUfUSs
Connection established to: QmRbXUDxrme7hRHJX4haYVnHcSWEcfgiD5QLuH1HyUfUSs
```

At this point, the nodes have connectivity but not a lot else. They don't actually *do* anything.

## Message Bus

## REST Endpoint

## Conclusion
