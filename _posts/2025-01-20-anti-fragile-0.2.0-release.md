---
layout: post
title:  "Anti-Fragile Panda (0.2.0)"
subtitle: "Now with full offline-first networking support"
author: glyph
---

Today we're happy to announce a new release of p2panda, one which offers exciting improvements to our networking and discovery capabilities.

## More "offline-first", please!

A major design requirement of p2panda is the ability to operate in an "offline-first" manner. The services we offer should be tolerant of volatile network environments, with the capacity to self-heal when interfaces and connections go down or become available. This stands in contrast to many contemporary networked applications and services which both expect and demand continuous connectivity. 

Our collaboration with several GNOME developers on a local-first GTK text editor (working title: [Aardvark](https://github.com/p2panda/aardvark)) quickly brought to light shortcomings in our "offline-first" resilience; data sync and live-mode would fail to recover after more than 30 seconds of lost connectivity and our mDNS discovery service would throw an error if no interface was available on startup. We've made it a priority to rectify these issues and believe that our `0.2.0` release makes significant improvements.

**Bidirectional Sync**

Firstly, we've refactored our log sync protocol to be bidirectional, meaning that both peers exchange their log heights and data in a single session. Our previous implementation required each peer to initiate a separate session for complete synchronisation. The change from a pull-based to push-based approach means that one peer can reset their sync state (e.g. after a network disconnection and reconnection) and initiate a session, resulting in both peers "catching up" on past data.

**Live-mode State Reset**

p2panda currently relies on gossip overlays for network-wide topic discovery and "live mode" (ie. broadcast of published data to all peers interested in a given topic). We discovered that these overlays break down after extended loss of connectivity. Our new release rectifies this issue by resetting live-mode state and re-entering gossip overlays when connectivity is regained.

**mDNS Retries**

What happens if you register an mDNS discovery service and then start your p2panda-powered network node when your device networking is disabled? In p2panda `0.1.0` this would result in a panic, as no socket was available to be fully configured and bound. Not good! We've refactored the service to (re)bind the socket as needed, ensuring that mDNS discovery can (re)start when interface changes are detected.

## Until Next Time

At the end of the month we'll be heading to Switzerland for [P2P Basel](https://p2p-basel.org/), a weekend workshop which "bring[s] together researchers and software builders to share insights and collaborate towards the sound and sustainable development of efficient eventually-consistent (offline-first) peer-to-peer systems". We can't wait to spend time with new and old friends alike!

Other than that, we're working hard on the [autonomous coordination toolkit](https://github.com/toolkitties/toolkitty) and designing our group encryption and access control systems.

We're looking forward to hearing from you as you try out p2panda `0.2.0`! Please consult the [CHANGELOG](https://github.com/p2panda/p2panda/blob/main/CHANGELOG.md) for a full list of changes.

Remember to subscribe to our [RSS feed](https://p2panda.org/feed.xml) for new upcoming posts on our website or follow us on the Fediverse via [@p2panda@autonomous.zone](https://autonomous.zone/@p2panda) to hear more about upcoming events or news!
