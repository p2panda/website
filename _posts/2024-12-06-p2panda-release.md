---
layout: post
title:  "2024 Updates and Our New Release!"
subtitle: "So much has happened this year and we're so excited to finally tell you about it!"
author: adz
---

So much has happened this year and we're so excited to finally tell you about it!

p2panda has been undergoing some changes; it's lighter, more hackable and much more modular. While this rewrite felt scary at times, we are very proud and excited about the outcome we're releasing today. This post is intended to shed light on our decision and convey why we believe the new p2panda will be useful for the wider "local-first" community!

## The "new" p2panda

We've released our [new crates](https://github.com/p2panda/p2panda) today and with this we're taking a new approach with p2panda:

**Modular and interoperable**

p2panda wants to lower the barrier for developers to build modern, privacy-respecting and secure local-first applications for mobile, desktop or web. We've learned that such a toolkit shouldn't come at the cost of a highly abstract "monolithic" API or framework. With that in mind, our new version aims to be as modular as possible—allowing projects the freedom to pick what they need and integrate it with minimal friction. Our networking layer offers a great example of a module which should be useful for many different projects. We believe this approach contributes the most to a wider, interoperable p2p ecosystem which outlives "framework lock-in".

For projects seeking a more fully-integrated system, it's possible to stack our modules into one efficient, stream-based pipeline providing p2p networking, sync, discovery, gossip, blobs, authentication, ordering, deletion, multi-writer, access control, encryption and so on.

**Data-type agnostic**

Some of our new building blocks, such as the sync, discovery and networking layers, do not require any custom p2panda data types. This makes it possible to bring your own data types and develop your protocol on top. We're planning the same for our group encryption implementation which is set to be released next year.

**Support any CRDT or application data**

Previous versions of p2panda came with their own approaches to CRDTs and schema validation. While we still believe this is great for future high-level modules, we wanted to offer you the option of combining all p2panda modules with Automerge, Yjs or any other CRDT of your choice. Of course not every application needs a CRDT and at the end of the day the payload is just "raw bytes".

**Re-use as much existing technology and well-established standards as possible**

We’re using existing libraries like iroh and well-established standards such as BLAKE3, Ed25519, STUN, ICE, CBOR, TLS, QUIC, UCAN, PlumTree, HyParView, Double Ratchet and more - as long as they give us the radical offline-first guarantee we need.

**Radically distributed and compatible with any post-internet communication infrastructure**

We want collaboration, encryption and access-control to work even when we can’t assume any sort of stable connectivity over an extended period. p2panda is "broadcast-only" at it’s heart, making any data not only offline-first but also compatible with post-internet communication infrastructure, such as shortwave, packet radio, Bluetooth Low Energy, LoRa or simply a USB stick.

**Append-only logs are great!**

We've chosen to remove Bamboo as a core data type from the p2panda protocol; we realised that we weren't making full use of the features it provides. Still, the new p2panda data type is again an append-only log, just much lighter and more flexible.

Logs are very efficient data types for exchanging data over challenged communication infrastructure and give developers something they will always need when building a distributed application: ordering (in other words, a knowledge of "what happened before / after or at the same time as x"). Our new data type also includes exciting features and optional extensions, such as writing to multiple logs at the same time, pruning (automatic removal of unused data), prefix-based deletion (delete multiple logs at the same time with a single tombstone) and fork-tolerance. Some of these features are rooted in exciting new research and we're happy to be sharing more about them in a future blog post.

As already mentioned, not all modules require you to use p2panda data types, but if you want the most features, they are ready with all sorts of extensions you can pull in for your application's needs.

## So much news!

**Collaborations with GNOME and HIRO**

While working on the new version of p2panda we also embarked on collaborations with two very different teams. We've been exploring code, UX and UI patterns for GTK-based applications with a group of developers from GNOME and together will release the first GTK-based, collaborative, local-first text editor. Our second collaboration has been a project developed together with HIRO, a company based in the Netherlands. Together we designed and implemented a solution named "rhio" to sync large files and messages between micro data centers in a fully distributed manner.

**Autonomous Coordination App**

This autumn we started working on an autonomous coordination toolkit with a new team of six people, supported by the [Innovate Creative Catalyst Programme](https://iuk-business-connect.org.uk/programme/creative-catalyst/) in UK. The goal of the project is to develop a mobile and desktop app for collectives, organisers and places to share resources and organise events in a shared calendar. This tool has been a [very old goal](https://media.ccc.de/v/36c3-10756-p2panda) of p2panda and it feels incredibly special to have the release of the first prototype scheduled for Spring of 2025!

![Ongoing development of the resource sharing and event organising app with p2panda.](/assets/images/very_large_phone.jpg)

> Ongoing development of the resource sharing and event organising app with p2panda.

## What's next?

p2panda is a very multifaceted project: We maintain our crates, apply for grants, design protocols and do research in radically distributed data-types. We organise community events and write peer-to-peer applications with our friends and collaborators. There's a lot coming up.

**Improve!**

This is our first release of the new p2panda version and we will surely learn more about it's APIs and user requirements in the upcoming months. Our goal is to reach a stable API but for now we need to expect breaking changes as we're adjusting.

**Group Encryption and Capabilities**

Next year we'll also be working on an [NLNet](https://nlnet.nl/) NGI Zero Entrust grant to integrate [UCAN](https://github.com/ucan-wg/spec)-based access control and secure group encryption with Post-Compromise-Security and optional Forward-Secrecy, based on research into [decentralised secure group messaging](https://dl.acm.org/doi/10.1145/3460120.3484542) algorithms. Our plan is to implement these as Rust modules which you can pull into your application, independent of p2panda, your choice of data types or networking stack. The DCGKA algorithm we'll be implementing is essentially Signal's Double Ratchet Algorithm with PCS and FS, made fit for offline-first use.

Together with researchers we'll be publishing our work on fork-tolerant and prunable append-only logs, hopefully in the form of another blog post or even a paper.

**App releases!**

For next year we will be releasing the GTK-based text editor in collaboration with GNOME and the first version of the autonomous coordination app (name still pending) and hopefully organise a festival with it sometime :-)

## Get involved

Please subscribe to our [RSS feed](https://p2panda.org/feed.xml) for new upcoming posts on our website or follow us on the Fediverse via [@p2panda@autonomous.zone](https://autonomous.zone/@p2panda) to hear more about upcoming events or news!

Our crates are ready to be played with and we are more than curious to hear about your ideas or feedback.

We are very excited to be hearing from you!

![Tired but happy p2panda team: adz, sam and glyph in London](/assets/images/adz_sam_glyph.jpg)

> Tired but happy p2panda team: adz, sam and glyph in London.
