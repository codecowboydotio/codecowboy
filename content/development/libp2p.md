+++
title = "Libp2p experiments with NGINX Unit"
date = "2021-12-30"
aliases = ["libp2p"]
tags = ["libp2p", "control plane"]
categories = ["software", "dev"]
author = "codecowboy.io"
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
Libp2p can be used to write distributed applications, as I mentioned above. In this case I'm going to demonstrate by writing a distributed control plane component for NGINX Unit that syncs configurations.

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

The first thing to do is the import all of the required libraries - you'll also need to **npm install** them as well. 
Note that I'm using **express**, **request** and **body-parser** - each of these is used as part of my REST endpoint later on.

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

This could be a topic in its own right, which I may get to as I extend this code.

Libp2p has a robust security framework in that it uses PKI to identify nodes. Each node has a certificate and public / private key pair. These can be auto generated or you can import your own. 

In addition to this the transport layer has connection encryption.
In my case I am using the NOISE module to provide this. This is a libp2p module that is based on the NOISE protocol. This is a well known protocol used by a number of projects, including libp2p.

More information on the NOISE protocol can be found [here](http://www.noiseprotocol.org/)

In short - using the noise protocol as part of my configuration means that I can encrypt the communication channels end to end as part of the library I'm using.

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

## Run the code
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

At this point, the nodes have connectivity but not a lot else. They don't actually **do** anything.

## Message Bus

Libp2p has a message bus that you can use to pass messaging between the nodes. 
This is important when you're trying to get messages from one node to all nodes. 
Note that the implementation that I am using is peer to peer and does not have a central hub.
All nodes listen to all other nodes for messages.

The simplest incarnation of this code is below:

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

  // now the node has started we can do our pubsub stuff

  const topic = 'news'
  node.pubsub.subscribe(topic)
  console.log('pubsub subscribe')

  node.pubsub.on(topic, (msg) => {
    console.log(`received: ${uint8ArrayToString(msg.data)}`)
  })

  setInterval(() => {
    var today = new Date();
    var date = today.getFullYear()+'-'+(today.getMonth()+1)+'-'+today.getDate();
    var time = today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds();
    var dateTime = date+' '+time;
    console.log('publishing: ', dateTime)
    node.pubsub.publish(topic, uint8ArrayFromString(dateTime))
  }, 1000)

})();
```

This code will emit a message onto the message bus using the current date and time. 
All other nodes will print the current date and time with "received" to their console.

### The magic part
The magic part of this is the following code.
Essentially, this sets up a topic called **news** and subscribes the node to it.

Then it sets up a listen event that logs out the message that is received. 

You will note two things, the data needs to be encoded / decoded as a uint8Array and the **msg** object contains a lot more information than just the pub sub message.

```javascript
  const topic = 'news'
  node.pubsub.subscribe(topic)
  console.log('pubsub subscribe')

  node.pubsub.on(topic, (msg) => {
    console.log(`received: ${uint8ArrayToString(msg.data)}`)
  })

  setInterval(() => {
    var today = new Date();
    var date = today.getFullYear()+'-'+(today.getMonth()+1)+'-'+today.getDate();
    var time = today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds();
    var dateTime = date+' '+time;
    console.log('publishing: ', dateTime)
    node.pubsub.publish(topic, uint8ArrayFromString(dateTime))
  }, 1000)
```

### Message Object
The message object contains more information than just the message data. 
It also contains the topic that the message was received on, and which node sent the message. 
There is also a sequence number and a signature for validation of the message and the endpoint.

We will simnply be using the message data and the node that sent the message.

```Bash
My Node ID:  QmeWfbAN9W7hZ5x6zkfNuixtyN9vg9cUTXkYdXfVXzNcwf
pubsub subscribed to topic successfully
received:  {
  receivedFrom: 'QmeWfbAN9W7hZ5x6zkfNuixtyN9vg9cUTXkYdXfVXzNcwf',
  data: Uint8Array(32) [
     73,  32,  97, 109,  32,  97,  32, 108,
    105,  98, 112,  50, 112,  32, 110, 111,
    100, 101,  44,  32, 104, 101,  97, 114,
     32, 109, 101,  32,  82,  79,  65,  82
  ],
  topicIDs: [ 'news' ],
  from: 'QmeWfbAN9W7hZ5x6zkfNuixtyN9vg9cUTXkYdXfVXzNcwf',
  seqno: <Buffer f0 1a f6 79 4d 3b 2e 54>,
  signature: <Buffer 45 14 c6 0c 52 20 33 e6 05 8f fb 26 dc 2f 54 44 74 c6 23 a9 9b 01 bb 76 2a 26 21 1a 78 e2 aa 91 13 1a 2e 47 9e b2 18 b5 fc ce 5c da c0 bd cf cd 92 43 ... 206 more bytes>,
  key: <Buffer 08 00 12 a6 02 30 82 01 22 30 0d 06 09 2a 86 48 86 f7 0d 01 01 01 05 00 03 82 01 0f 00 30 82 01 0a 02 82 01 01 00 d8 af 26 e0 7d 66 13 d5 27 6e 2e 55 ... 249 more bytes>
}
received data: I am a libp2p node, hear me ROAR
foo QmeWfbAN9W7hZ5x6zkfNuixtyN9vg9cUTXkYdXfVXzNcwf
```

### Running the code
Running the code now looks like this:
Each node will both publish and subscribe to messages on the bus.

```Bash
My Node ID:  QmWnjT4paQSozSUWH3DZ5RMbdFEdCQwbFNo911oyNs4qWx
pubsub subscribe
Discovered: QmWav8yx1Feaqg1frQjp1MFKtdNDa3spuuQkTMTBPZBDwm
Connection established to: QmWav8yx1Feaqg1frQjp1MFKtdNDa3spuuQkTMTBPZBDwm
Connection established to: QmWav8yx1Feaqg1frQjp1MFKtdNDa3spuuQkTMTBPZBDwm
Connection established to: QmWav8yx1Feaqg1frQjp1MFKtdNDa3spuuQkTMTBPZBDwm
Connection established to: QmWav8yx1Feaqg1frQjp1MFKtdNDa3spuuQkTMTBPZBDwm
publishing:  2021-12-31 14:31:32
received: 2021-12-31 14:31:33
publishing:  2021-12-31 14:31:33
received: 2021-12-31 14:31:34
publishing:  2021-12-31 14:31:34
received: 2021-12-31 14:31:35
publishing:  2021-12-31 14:31:35
received: 2021-12-31 14:31:36
publishing:  2021-12-31 14:31:36
```

You can see in the output above that the node starts, just as before. It establishes a connection to the other nodes in the network, and listens to messages. The node is subscribing to messages on the bus, but is also publishing messages onto the bus.

Each new node that starts with the same code, will publish and subscribe messages in the same way.


### What have we got at this point?
At this point, we have a piece of code that can be run on multiple nodes that can do the following things:

1. Get a node address
2. Discover other nodes on the networks
3. Subscribe to a message bus topic
4. Listen for messages from other nodes on that topic
5. Process messages from other nodes.

It's certainly very interesting but not particularly useful at this point, but it's a decent scaffold that we can begin to use for other purposes.

## What now?
What now is a very good question. 

In my case I want to synchronise configuration among a number of servers. 
Each of these servers has a REST API that allows me to get configurations and put configurations back onto them.

It's going to work something like this:

Apply update to server.
Hit REST endpoint on server.
REST endpoint triggers a message over the message bus to say "I have a configuration update".
All other servers subscribed to that topic then pull the configuration from the server that says it has a configuration update.

It looks something like this:

![libp2p_config_flow.JPG](/images/libp2p_config_flow.JPG)

## REST Endpoint
Finally the single thing that ties all of this together is the REST endpoint that I have written.
This is an external interface for interacting with the peer to peer network that gets built.
I have used the node **express** module to provide an HTTP listener that responds to POST requests. 

```javascript
  app = express()
  port = process.env.PORT || 3000;

  app.use(bodyParser.json())

  app.post('/config', (req, res) => {
    var today = new Date();
    var date = today.getFullYear()+'-'+(today.getMonth()+1)+'-'+today.getDate();
    var time = today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds();
    var dateTime = date+' '+time;
    let ar_host_port = req.rawHeaders[1].split(":")
    console.log('publishing: ' + dateTime + ' ' + ar_host_port[0])
    res.end('Published config event to all other nodes');
    node.pubsub.publish(topic, ar_host_port[0])
  })
  app.listen(port, () => {
    console.log('app listeneing on: ' + port)
  })

```

The snippet above is a fairly conventional express app with the exception of two things.

```javascript 
    let ar_host_port = req.rawHeaders[1].split(":")
    node.pubsub.publish(topic, ar_host_port[0])
```

The first line pulls the IP address of the receiving host from the express request headers. 
This is useful to identify the IP address of the host that is sending the message onto the message bus.
This is sent as part of the message to all other nodes, and tells subscribed nodes where to get their configuration from.

If I run this code end to end it looks something like this:

```Bash
root@unit-1:~/libp2p-experiments/src# node mdns-unit.js
My Node ID:  QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
pubsub subscribe
app listeneing on: 3000
Discovered: QmPL7k2Pk6iCeV9DWx61vjHKFtAvPbMAHAyhEJcVMKyQ4a
Connection established to: QmPL7k2Pk6iCeV9DWx61vjHKFtAvPbMAHAyhEJcVMKyQ4a
Connection established to: QmPL7k2Pk6iCeV9DWx61vjHKFtAvPbMAHAyhEJcVMKyQ4a
Connection established to: QmPL7k2Pk6iCeV9DWx61vjHKFtAvPbMAHAyhEJcVMKyQ4a
publishing: 2021-12-31 10:49:55 10.1.1.151
```

Above you can see node number one starts and automatically discovers node number two.
Node one also publishes a message that contains the IP address of node one.

```Bash
root@unit-2:~/libp2p-experiements/src# node mdns-unit.js
My Node ID:  QmPL7k2Pk6iCeV9DWx61vjHKFtAvPbMAHAyhEJcVMKyQ4a
pubsub subscribe
app listeneing on: 3000
Discovered: QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
Connection established to: QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
Connection established to: QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
Connection established to: QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
received: 10.1.1.151 from QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
pulling config from QmeU7j8JNPhg3kxYWndC2GENRiErBigPsHhN7keEAx664G
Body that I got is:  {
  listeners: { '*:80': { pass: 'routes' } },
  routes: [ { action: [Object] } ]
}
------
{
  listeners: { '*:80': { pass: 'routes' } },
  routes: [ { action: [Object] } ]
}
{ success: 'Reconfiguration done.' }
```

The output above shows the node starting and discovering the first node.
Importantly, it also shows a message being **received** from the first node. 
The second node then connects to the IP address in message and pulls the NGINX Unit configuration from that node, and applies it to localhost.

You can see the message **Reconfiguration Done** - this is some debug output that I print to the screen. It is the response from the PUT request to the unit server that updates the configuration.

## Handling the REST request
The code block below handles the publish event from any other node.
This code will run when a message is received on the topic that the node is subscribed to.

The code does three things:
1. Receiving an event on the message bus.
2. Getting the configuration from my NGINX Unit server.
3. Processing the configuration and applying it locally.

```javascript
  node.pubsub.on(topic, (msg) => {
    console.log(`received: ${uint8ArrayToString(msg.data)} from ${msg.from}`)
    console.log(`pulling config from ${msg.from}`)
    let conf_url = 'http://' + msg.data + ':8888/config'
    request(conf_url, { json: true }, (err, res, body) => {
      if (err) { return console.log(err); }
      console.log('Body that I got is: ', body)
      const unit_config = body
      console.log('------')
      console.log(unit_config)
      // let's try to put the config locally
      request.put({
        headers: {'content-type' : 'application/json'},
        url: 'http://127.0.0.1:8888/config',
        json: unit_config
      }, function (error, response, bdy){
           console.log(bdy)
      }) //end request.put
    })
  })
```

## Complete code

The complete codebase that I used it below.
Some part of it are hard coded - like the port that the NGINX Unit configuration service runs on (8888).
These are easily changed, and in a later update I will make these environment variables.


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

  // now the node has started we can do our pubsub stuff

  const topic = 'news'
  node.pubsub.subscribe(topic)
  console.log('pubsub subscribe')

  node.pubsub.on(topic, (msg) => {
    console.log(`received: ${uint8ArrayToString(msg.data)} from ${msg.from}`)
    console.log(`pulling config from ${msg.from}`)
    let conf_url = 'http://' + msg.data + ':8888/config'
    request(conf_url, { json: true }, (err, res, body) => {
      if (err) { return console.log(err); }
      console.log('Body that I got is: ', body)
      const unit_config = body
      console.log('------')
      console.log(unit_config)
      // let's try to put the config locally
      request.put({
        headers: {'content-type' : 'application/json'},
        url: 'http://127.0.0.1:8888/config',
        json: unit_config
      }, function (error, response, bdy){
           console.log(bdy)
      }) //end request.put
    })
  })


  app = express()
  port = process.env.PORT || 3000;

  app.use(bodyParser.json())

  app.post('/config', (req, res) => {
    var today = new Date();
    var date = today.getFullYear()+'-'+(today.getMonth()+1)+'-'+today.getDate();
    var time = today.getHours() + ":" + today.getMinutes() + ":" + today.getSeconds();
    var dateTime = date+' '+time;
    let ar_host_port = req.rawHeaders[1].split(":")
    console.log('publishing: ' + dateTime + ' ' + ar_host_port[0])
    res.end('Published config event to all other nodes');
    node.pubsub.publish(topic, ar_host_port[0])
  })
  app.listen(port, () => {
    console.log('app listeneing on: ' + port)
  })

})();
```

## Conclusion
Libp2p is a fantastic library that allows peer to peer connectivity and communication between nodes in a secure fashion using verifiable PKI and encryption.

In my case it's been a fantastic journey of discovery (no pun intended) to figure out how to make libp2p work as a distributed control plane. The benefits of this are obvious in that if every node *is* the control plane, then there are fewer failure domains. 

The thing that I like the most about this is that it's all done in about 104 lines of code, and I'm fairly sure that I can make this smaller again by making my code a bit tighter and removing some of the obviously **development** heavy debugging from my code.

Look out for the next edition where I walk through using the libp2p router component to cross layer three networks.
