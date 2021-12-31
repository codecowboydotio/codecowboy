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

## Message Bus

## REST Endpoint

## Conclusion
