--- 
layout: post 
title: "Sync in diverse networking contexts" 
subtitle: "Findings from Dash Router work and improved sync APIs" 
author: sam 
---

We're mid-way through our [Prototype Fund](https://www.prototypefund.de/en/) supported collaboration with the folks from Dash Chat, the goal of which is [Dash Router](https://www.prototypefund.de/en/projects/dash-router); a delay-tolerant relaying protocol for mesh networks with built-in store-and-forward properties and eventual delivery guarantees. The responsibilities of the p2panda team have been to provide a user friendly API for "spaces" (pub-sub with group encryption) and a mesh-suitable sync protocol which supports store-and-forward. In this article we report on the process and output of our work on providing the required sync primitives and APIs.

## Architecture design

Our starting point for designing the architecture was grounded in our existing ideas around the "Walkaway Stack". We try to clearly separate event "delivery" from event "processing"; doing so allows for the p2p protocol and domain logic to be agnostic of the underlying transport and general networking layers. One substantial challenge in designing this particular architecture was understanding if the mesh routing strategy could, or indeed should, be agnostic to the transport and approach to sync, and being aware of the trade-offs which came with different approaches. After much research, design and discussion we came to the conclusion that Dash Router should be a stand-alone system taking care of networking, routing and sync. This was informed by the knowledge that mesh routing is a deeply nuanced problem space, and any approach will need to be verified against simulations and real-world testing and adjusted in an iterative manner. Separating concerns too early would remove certain introspection and configuration options which might be later needed. This design neatly fits with the existing p2panda stack though, as Dash Router becomes a fully self-contained delivery strategy that provides a simple external event stream which can be fed into the existing p2panda event processing pipeline.

## Easy-to-use Sync API

Our architecture decision meant that the sync protocol implementation would happen inside Dash Router using existing p2panda log-based sync protocols. As p2panda already designs for "post-internet" networking the protocols themselves didn't need any adjustments, however it soon became clear that the existing APIs were not designed with the composable approach we needed in mind, and thus were hard for external developers to work with. We decided to focus on providing a sync API which could easily meet the needs of some common networking topologies; Sneakernet, Unicast w/ eager push and Broadcast. We're really happy with the result of this work, you can check-out the [extensively documented new APIs](https://github.com/p2panda/p2panda/blob/main/p2panda-sync/src/api.rs) and accompanying examples for use over [sneakernet](https://github.com/p2panda/p2panda/blob/main/p2panda-sync/examples/sneakernet.rs), [unicast](https://github.com/p2panda/p2panda/blob/main/p2panda-sync/examples/unicast_w_live_mode.rs) and [broadcast](https://github.com/p2panda/p2panda/blob/main/p2panda-sync/examples/broadcast.rs) transports.

## MemoryLAN

An important piece of research which took place during research was understanding and making a [reference Rust implementation](https://github.com/p2panda/memorylan) of MemoryLAN "Local Area Content Replication Mesh" by Christian Tschudin. Although Dash Router may not use the implementation itself, exploring the approach and terminology was of much help in understanding and discussing the general problem space. We are still very excited by the "shared cache" approach the design expresses, and hope to perform simulations and real-world tests using it in the future.

## What's next?

Hacking now starts towards a proof-of-concept Dash Router implementation, which will be integrated into Dash Chat and tested on a LAN network. Finally, we hope that the new sync APIs and examples can inspire someone to experiment with their own delivery layer implementations, and would love to hear about your adventures!