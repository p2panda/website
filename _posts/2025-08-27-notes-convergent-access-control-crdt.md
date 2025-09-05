---
layout: post
title: "Notes on building a convergent, offline-first Access Control CRDT"
subtitle: "General learnings on CRDT design and how they can be integrated into applications"
author: adz
---

p2panda recently published the first release of [`p2panda-auth`](https://crates.io/crates/p2panda-auth), a convergent, offline-first CRDT ([Conflict-free Replicated Data-Type](https://mattweidner.com/2023/09/26/crdt-survey-1.html)) which helps managing members, admins or moderators in a group and giving them permission to sync, read or change application data. If you want to learn about the concrete implementation and design choices of `p2panda-auth` we have an [in-depth blog post](https://p2panda.org/2025/07/28/access-control.html) for you here.

This post is about our general learnings on building an **Access Control CRDT** followed by our exploration of patterns for how application data can be “authorised”. This includes **Application Integrations** and **Combination with Key Agreement Protocols**. These patterns can be very different based on the individual application’s requirements (and we surely didn’t cover a lot of other approaches). There is no straight-forward answer or single solution - so here is an attempted “summary”!

## Do you need eventual consistency?

If we can trust a single server to authorise edits from different users we can simply rely on the “linear history” of access control- and data changes. The changes are “serialised” in the order the server received them.

![Centralised server serializing all permission- and app-data changes](/assets/images/250827-centralised-server.png)

> Centralised server serializing all permission- and app-data changes

We can apply a similar rule in peer-to-peer systems as well: As soon as the peer learned that a permission was revoked (the user is not allowed to change the data) we reject any future data from now on. Every peer can manage that “authorisation state” themselves and act accordingly from their perspective.

![Authorisation in peer-to-peer without eventual consistency](/assets/images/250827-without-eventual-consistency.png)

> Authorisation in peer-to-peer without eventual consistency

This example shows how both peers “synced” the removal of Owl’s permission at one point, but allowed different edits of Owl before doing so. Peer A will end up with a different app state than Peer B. In systems like that we can’t guarantee the “eventual consistency” of the application data. At least not without further work.

Imagine an often changing key/value database, if peers constantly overwrite the values with new data (in a “last-write-wins” fashion), it maybe doesn’t matter if they are briefly “out of sync” until they converge to the same state again.

In p2panda we believe that this is a very valid option for building many peer-to-peer applications. The trick will be to figure out patterns (once again) to identify when and how which guarantee is required for which application.

For a computer game that might not be so nice though as maybe Owl ended up making an extra move gaining more points for Peer A, while Peer B has a different game state.

The answer is really: it depends!

How would we introduce eventual consistency here? We would need to detect concurrent and conflicting changes, for example a user changing data while they concurrently have been removed and be able to retroactively handle the “unauthorised” edits.

An access control system doesn’t need to be complicated when applications don’t require eventual consistency guarantees from it or if they can model them outside of that scope. Simple [Capability-based Tokens](https://en.wikipedia.org/wiki/Capability-based_security) with an expiry date can be enough to account for such applications.

In p2panda we want an access control solution which can account for all sorts of application needs, including the guarantee to “converge” to the same application state based on the given access control, with all the bells and whistles we need around concurrency and conflict resolution. The answer is to build a “convergent”, offline-first Access Control CRDT and we will show later how we can integrate it into application data, with different patterns around eventual consistency, moderation and encryption.

First let’s build that CRDT.

## Building a convergent, offline-first Access Control CRDT

There are different strategies to “authorise” someone to do something in an application. In our setting we would like users to create groups. These groups provide a context to manage higher-level key agreement CRDTs for group encryption and are “composed” on top of the Access Control CRDT. A group creator is able to add and remove other users and give them “access rights”, for example write or admin access.

This is similar to an [Access Control List](https://en.wikipedia.org/wiki/Access-control_list) (or ACL) which is a bit like a guest list of a party: If you’re on the list you are allowed to enter, if not - you can’t. Every peer manages such a list and checks it to ensure that received user actions are permitted.

This is different from building a convergent capability-based CRDT, like in [Keyhive](https://www.inkandswitch.com/keyhive/notebook/), where users form “delegation” chains to permit access. However, the CRDT parts are fairly similar.

### Dependencies & out of order handling

Independent of the application data we know one thing for sure: We want access control to *always* be consistent! Every peer should be able learn about the latest and same status of the access control system at one point.

This doesn’t seem so hard to achieve at first sight: Every peer eventually receives the “user got removed” operation and converged to that state, knowing they need to reject that user’s data from this point on.

Things can still get a little bit tricky though: What if we receive the operations out-of-order? We receive the user’s removal before they were even added? We need a way to describe “dependencies” to understand if we’re missing operations.

For this we give every operation a unique identifier, for example a hash. Now operations can “declare dependencies” by mentioning a list of operation IDs which they consider important to be “processed before”. Peers who receive an operation can now understand if they’re still missing other operations before they can go on.

This structure of dependencies forms a [Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) (DAG). Our favourite graph for many peer-to-peer problems! <3

![Graph describing dependencies](/assets/images/250827-dependencies.png)

> The “edges” of the graph describe the “dependencies” of every “node”.

Peers can now easily reason about what they are missing and process things only if they know that they have seen every dependency of that operation.

If we look closer we can see that the “processed” operations are effectively a linearised, [topologically ordered](https://en.wikipedia.org/wiki/Topological_sorting) sequence after their dependencies have been checked.

Now we can at least be sure that we haven’t “missed” anything. If we choose our hashing algorithm wisely we can make sure that it is impossible to “guess” an operation id because they are simply too long and “too random”. Like this, peers can only refer to an operation if they really learned about it before. This gives us the guarantee of **Integrity**.

### Ordering

Another property we have from DAGs is that we can start to reason about the “order” of operations.

In our decentralised systems we can’t really reason about the “exact” order of concurrent operations as we don’t have one single source of truth which orders events for us (like a centralised server or a consensus-based, [single-ledger blockchain](https://en.wikipedia.org/wiki/Blockchain#Decentralization)). This is why DAGs are perfect to describe this “partial ordering”. Partial order is a set of operations where we can’t always compare two entries with each other and know which one was created first. When we can’t compare them directly we only know they occurred *at the same* “logical” time.

![Partial Order](/assets/images/250827-partial-order.png)

> We don’t know exactly in which order C and D took place here, we only know that these have been “concurrent” operations as both of them didn’t know about each other while they got created.

### Authorisation

How can we know that a member was really allowed to add someone to the group?

We need to make sure that new operations can only be applied to the graph if previous operations authorised them. Like this we can form a sort of “trust chain” or proof and trace back the “logical” operations until the beginning and check on every step if they got authorised to do that action.

![Authorisation Chain](/assets/images/250827-authorization-chain.png)

> Owl is allowed to add Pig to the group because we can see that Owl was made an Admin by Sheep before and that Sheep is an admin because they created the group before

If we sign every operation on top with a [cryptographically-secure signature](https://en.wikipedia.org/wiki/Digital_Signature_Algorithm) we can make sure that the author of this change can prove their identity. This gives us the guarantee of **Provenance**.

### Concurrency

By looking at a DAG we can reason about the partial order and concurrent operations easily. How do we identify concurrent operations programmatically?

In a DAG we can use a traversal technique to identify all concurrent operations from the perspective of a single “target operation” by moving from that target to all other reachable nodes in the graph, in depth-first order. We mark all visited nodes as “successors” of that target.

Next we *reverse* all edges and do the same traversal again. All visited nodes can now be marked as the “predecessor” of the target. Like this we get a set of all operations which are neither predecessors nor successors of the target operation which means that they have been concurrent to it.

We need to repeat this process for every node in the graph. Each time we detect a set of “concurrent nodes” we recurse down each of them, repeating the same process again but merging the outcome as one “concurrent bubble”. Bubbles which only contain one node are ignored.

![Calculating concurrent bubbles I](/assets/images/250827-concurrent-bubbles-1.png)

Traversing the graph with this technique gives us a list of all concurrent bubbles in it. At this stage it is not necessary to account for “nested” bubbles. It is sufficient to consider them as one single entity. 

![Calculating concurrent bubbles II](/assets/images/250827-concurrent-bubbles-2.png)

This is arguably a very involved way to compute concurrent bubbles in a graph and using [Lamport Timestamps](https://en.wikipedia.org/wiki/Lamport_timestamp) can be more efficient. However, they do not give us the integrity guarantees of the hashes, so implementations will probably end up having both systems next to each other.

### Conflicts

Where things get the most tricky is on concurrent group changes which might be in conflict with each other. For example: What happens if someone wants to add a member to the group while they are concurrently being removed?

Since we can now detect concurrent operations or “bubbles” we can apply rules for merging conflicting changes.

There are very different approaches on how to handle these conflicts, all coming with different advantages and disadvantages:

* **Seniority ranking:** “older” members win over concurrent removals. Prone to [Sibyl attack](https://en.wikipedia.org/wiki/Sybil_attack) so further mitigation is required
* **Remove both:** Solution for byzantine scenarios where a group was “infiltrated” but it might end up without admins so the group needs to recover by starting anew
* **Decide “by higher hash”:** Removal operation with “higher hash” value wins deterministically but “randomly”. Operations can be adversarially chosen to “win”

Which strategy to pick should be decided by a treat model and how well it can be communicated to the users. Which conflict resolution strategies are working well for Authorisation CRDTs and their users is still to be explored.

### Consensus & Finality

Whenever a peer publishes an operation in the DAG we can use their “dependency” pointers as proof that they have “acknowledged” the state up until that point in the graph.

Since the number of members is known in the group at this point we have a fixed bound of how many acknowledgements we need to reach “finality” or consensus on that current state.

This is useful for a whole range of improvements to our Auth CRDT:

1. We can explore pruning or compaction techniques to remove or “compress” parts of the history of the DAG
2. We can reason about what a peer has “missed” when they have acknowledged something. With that knowledge we might know something they didn’t, so we can “forward” that missing information to them. This gets especially interesting to account for concurrent changes in key agreement protocols, like `p2panda-encryption`
3. We can “lock in” the group state from this moment on and agree on finality: Every change which will be applied into that “past” will be considered **byzantine behaviour** and illegal. This “fork the past” we also call **equivocation** and consensus protocols like that can help us to detect and mitigate them

![Finality through consensus](/assets/images/250827-finality.png)

## Integration with p2panda

To build CRDTs based on DAGs we already have all the tools we need inside of our `p2panda-core` crate. Our `Operation` core data type gives us:

* Hashing functions to derive an identifier from the operation and guarantee Integrity
* Digital Signature Scheme for each operation, to guarantee Provenance
* Extensions to declare dependencies and partial order
* Append-only log structure per author to detect forks within a log; this can be optionally used to build byzantine fault tolerance on that layer

Additionally we provide ready implementations in `p2panda-stream` to efficiently order incoming operations based on their declared dependencies and keep them around (unloaded into a database) until “ready”.

With operations and the orderer we can now work with partially-ordered, dependency-checked (handling out-of-order arrival), authenticated and linearised streams of operations.

This is a basic building block to build powerful, authorised CRDTs or other new data types on top of. With `p2panda-auth` we’re introducing our first **eventually-consistent / convergent, authorisation CRDT**  which can be easily combined with p2panda operations and our orderer.

## Integration with applications

Different applications will have different requirements around integrating an Authorisation CRDT.

In any case we need some sort of “Events API” for the Access Control layer to inform applications about any group changes. In p2panda we also give the application information about *when in logical time* this event took place. This is the ID of the operation in the graph. For removal events we also mention the set of operations which potentially have been created by the removed member concurrent to their removal.

![Application Events API](/assets/images/250827-app-event-api.png)

With all of this information, group events and application messages, an application has everything to deal with authorisation changes. Developers will need to reason about the eventual consistency guarantees in their applications, depending on the kind of data, how content is “moderated” and how it is represented to the users.

The aim is to come up with “patterns” describing different use-cases and concrete examples in the future which hopefully will make these decisions easier.

We’ve already talked heaps about eventual consistency in the beginning of the blog post, so this might feel familiar:

**“Weak” / No eventual consistency:** As soon as the application is informed about a member’s removal it simply starts to exclude future messages authored by the member from that point on.

This approach is very simple and doesn’t guarantee eventual consistency. Peers might pick different points from where they’ve started to exclude messages from the removed member.

Depending on the application data that might not be a problem though, either because consistency is dealt with on a higher level or because the application doesn’t care.

Consistency could be reached again, for example, by using a key/value store with last-write-wins logic which converges to the same state as soon as another member overwrites the entry with a new value.

There’s no “automatic moderation” taking place, as in members would need to edit the data or remove messages manually etc.

**Guaranteed eventual consistency:** As soon as the application gets informed about a member’s removal it automatically removes the data from the point where the group changed, including concurrent messages.

For simple data types, like chat messages or “social media” posts etc. it will be easier to remove every data type associated with that concurrent operation, while for Text CRDTs we might need to “re-play” the operation graph from the removal point on and filter out all concurrent operations during materialisation.

With this “re-basing” or “re-playing” approach we can guarantee eventual consistency for complex data.

For moderation we can be sure that no messages will be displayed concurrent to the removal; anything which took place before needs to be manually removed or edited.

![Application Example with complex data-type](/assets/images/250827-app-example-1.png)

> First example shows a more “complex” text CRDT with an “invalid” change which needs to be retroactively removed. To remove the concurrent change of Panda we need to go back in time and “re-play” the changes to the text, without the concurrently removed operations. The state will be “re-materialised”.

![Application Example with simple data-type](/assets/images/250827-app-example-2.png)

> Second example is simpler, as we don’t need to re-do anything. We can simply remove the concurrently created post based on the Operation IDs we learned about from the removal event.

## Integration with key agreement protocols

When integrating an Authorisation CRDT with key agreement protocols for group encryption (for example `p2panda-encryption`) we have to be aware of some concurrency edge-cases which might lead to (accidentally) leaked secret keys.

This is a little excursion into [Key Agreement protocols in decentralised, offline-first systems](https://p2panda.org/2025/02/24/group-encryption.html) but it also shows how the composition of different convergent data-types can lead to interesting problems we need to think about.

We define our key agreement protocol in a way where on every “Add” Operation the newly introduced member learns about the secret keys for that group so they can decrypt other member's messages or encrypt new messages towards the group.

In a scenario where Panda creates a group with Panda and Bear inside, Bear concurrently adds Owl to the group while Panda removed Bear. We can resolve this “conflict” easily with our Authorisation CRDT but unfortunately the secret keys might have been leaked to Owl without Panda being aware of it!

![Leaking Keys Scenario](/assets/images/250827-leaking-keys.png)

In a system with forward secrecy, such as p2panda’s “Message Encryption” Scheme, we would not run into problems as Owl will not be able to decrypt any previously created messages. However, if we use encryption systems with only Post-Compromise security, like p2panda’s “Data Encryption” Scheme, Owl will now be able to decrypt all previously created data, even if they’ve only been very “briefly” in the group from Bear’s perspective!

To avoid this scenario in PCS-only encryption systems we need to ask for consensus from the group *before* we hand over secret keys to new members. We can achieve this with a form of acknowledgement protocol (similar to what we’ve described previously) and only allow sharing secrets *after* every removal has been acknowledged by everyone or at least a majority of the group.

## Patterns everywhere!

This was it for now for our little excursion into convergent data-types! We hope that it inspired you to see a range of "tricks" one can do with them to make them work in peer-to-peer environments.

It is still too early, but it makes us hopeful for a future where we will slowly converge to re-usable "patterns" which can be applied to all sorts of problems, apart from access control solutions. Across the p2panda stack we can slowly see those "repetitions". It surely helped to have developed all data types independently from each other, outside of a monolithic all-in-one solution as it forced us to be very clear about the common interfaces they share.

Maybe one day it will be very easy to "compose" them with other solutions outside of the p2panda universe because these patterns, terminology and requirements around convergent data types become well-understood and known? Imagine combining the access control data type with someone else's work to do efficient pruning or another one's code to detect and mitigate Byzantine behaviour? This is an quite exciting future for sharing code, progress and research across peer-to-peer projects!
