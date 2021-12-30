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

## Components 
My control plane will have a few conponents. 

- Peer identity
- Secure Connection
- Peer discovery
- REST endpoint

## Peer Identity

## Secure Connection

## Peer Discovery

## REST Endpoint

## Examples

## Conclusion
