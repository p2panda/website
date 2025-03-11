---
layout: post
title:  "Event Streams and Persistence (0.3.0)"
subtitle: "Bootstrap your network and listen to events"
---

We're excited to share a new release of p2panda, one which is primarily focused on our networking crate but also introduces SQLite persistence and several bug fixes.  

## Bootstrapping and Discovery

**Network Events API**

As a user of our `p2panda-net` crate, it's understandable that you might wish to have some degree of introspection into the peer-related events which are occurring. In this release we've introduced a means of subscribing to a stream of network events by calling `.events()` on `Network`. Events currently include those related to gossip (aka. "live mode"), sync (aka. data replication) and discovery. In this way, it's possible to learn when a sync session has started and finished, when a new peer has been discovered and when a connection has been establish with a new neighbour. We intend to add additional data to these events in the future, such as the amount of bytes synced and the duration of each session.

**Bootstrap Mode**

As our collaborators begin working more deeply with our modules, we've been discovering blindspots in our implementations and working on improvements. One such improvement is the introduction of a "bootstrap mode" for network peers. The bootstrap node is one which is started without knowledge of any peers; it serves as the entrypoint into the network for others. This ability is activated by calling `.bootstrap()` on the `NetworkBuilder` during network configuration. The [chat example](https://github.com/p2panda/p2panda/blob/main/p2panda-net/examples/chat.rs) in our `p2panda-net` repository provides a configurable CLI tool to play with various scenarios.

**Discovery Over the Internet**

With the bootstrap mode in place, we now have discovery of peers and topics over the internet. Using the power of [iroh](https://www.iroh.computer/) under the hood, all that's needed to join a network is the URL of a relay server (which must be the same amongst peers) and the public key of a bootstrap node or other online peer; the discovery process then occurs automatically.

## Filesystem Persistence

**SQLite Store**

You might be surprised to learn that up until this point we have only had in-memory persistence for operations and logs, the core data types of p2panda. This reflects our style when it comes to adding new features; we aim to coordinate our efforts to align with the needs of the apps being built by us and our collaborators. The need for filesystem persistence has recently arisen and so this release introduces a SQLite store. It can be accessed through `p2panda-store` with the `sqlite` feature flag enabled.

## Until Next Time

Next week we're on our way to the [Bidston Observatory Artistic Research Centre](https://www.bidstonobservatory.org) (BOARC) near Birkenhead, UK for a team working session! The core team (adz, sam and glyph) will join the Toolkitty team for a contentrated few days of app develpoment. [Toolkitty](https://github.com/toolkitties/toolkitty) is an autonomous coordination toolkit for collectives, organisers and venues to share resources and organise events in a collaborative calendar. We're getting closer to completing the prototype and can't wait to share in the months ahead!

We're looking forward to hearing from you as you try out p2panda `0.3.0`! Please consult the [CHANGELOG](https://github.com/p2panda/p2panda/blob/main/CHANGELOG.md) for a full list of changes.

Remember to subscribe to our [RSS feed](https://p2panda.org/feed.xml) for new upcoming posts on our website or follow us on the Fediverse via [@p2panda@autonomous.zone](https://autonomous.zone/@p2panda) to hear more about upcoming events or news!
