---
layout: post
title: "Access Control in Decentralised Systems"
subtitle: "Design and implementation overview of p2panda-auth"
---

Having just released the first version of our [p2panda-auth](https://crates.io/crates/p2panda-auth) crate, it seems like the right time to write about access control in decentralised systems. In the process, we'll share an overview of the system we've designed - as well as a discussion of some of the technical challenges involved in implementing peer-to-peer group management and access control. We're grateful to [NLnet](https://nlnet.nl/project/P2Panda-groups/) for supporting us in this work.

## System Requirements

Before diving into the details of the system we've implemented, let's first discuss why we might want an access control system? Broadly speaking, access control gives us a way to define **who** can interact with some specific data and **how** they can interact with the data. The "who" can be thought of as an actor; this might be a cryptographic keypair mapping to a single person or it may be a keypair which represents one of several devices controlled by a single person. An actor may even be a group which is itself composed of several other actors.

### Reading & Writing

The two most basic forms of interacting with data in an access control system are reading and writing. In the process of composing this blog post, for example, I'd like to give my teammates the ability to read _and_ modify the text while only allowing external advisors to read it. As another example, I may wish to have a music folder on my laptop which is shared between all of my other devices. Access control allows us to clearly define abilities and gives us a means of realising
these scenarios.

![Encrypted Data Read and Write](/assets/images/250716-read-write.png)

> Sloth has written a list of their favourite activities: sleeping, cuddling and climbing. Owl has read-access to the document; they can read the list that Sloth has authored but not make any changes. Cat, on the other hand, has write-access and decides to replace "climbing" with "meowing".

### Replication

In decentralised or peer-to-peer systems we also need to keep in mind that data travels between devices in a network. Since a direct connection to a particular peer is not always possible, we may wish for some intermediate peers to be able to assist with passing data through the network. Since those peers may be untrusted, or may simply not be the intended recipient of some data, it's useful to have a means of allowing the right to replicate without the write to read.

This is where encryption comes into the picture; it allows us to prevent unauthorised reading of data. The ability to read is thus mapped to the ability to decrypt. In systems with connection-based replication protocols we can rely on a seperate access control level to define who is allowed to receive data (even though that data is still encrypted). When connecting to a peer, before we begin sending any requested data, we first check whether that peer has pull-access. If not, we refrain from fulfilling the request.

![Encrypted Data Pull and Read](/assets/images/250716-pull-read.png)

> Sloth has write-access to a list of activities: sleeping, cuddling and climbing. Beetle only has pull-access to the list, meaning that they can receive and pass-on that data but cannot decrypt it for reading. Beetle replicates the data from Sloth and forwards it to Shark. Shark has read-access; they're able to decrypt the data and read the list that Sloth authored.

### Intuitive & Customisable

In the context of p2panda, we want access control to be intuitive for application developers to integrate into their software. In addition, we wish to allow for custom access conditions which can further constrain an actor's access level over some data, and we aim to be conservative in terms of meeting our security requirements. These aspects of our work will become clearer throughout the rest of this post.

## Implementation Approaches

There are several design approaches to meeting the requirements we outlined above. Here we briefly describe two such systems for maintaining and enforcing access to resources.

### Capability-Based Access Control

Capability-based access control systems rely on secure authorisation tokens and use delegation chains to verify which actors have access to any particular set of data. For example, I may issue a token granting read access to my photo-sharing folder with a relative. That token is then handed over as proof of access when my relative tries to read the folder. Such systems allow delegation of received access; an actor can pass on any received capability to other actors. [Meadowcap](https://willowprotocol.org/specs/meadowcap/index.html) from the Willow team is a good example of a pure capability-based access control system.

Delegations will most often include an expiry date, the expectation being that if access should be maintained a new token will be issued before the previous one has expired. Some systems include the ability to retroactively revoke a previously-delegated access. In such cases, all dependent delegations will also be revoked.

### Distributed Access Control Lists

An alternative approach to capability-based systems is the Distributed Access Control List (ACL). ACLs are commonly used to restrict filesystem access and access to resources on centralised servers. In such systems, a list is kept which maps an actor to an access level. In decentralised contexts, we also require the ability to collaboratively maintain and modify the list. To do so, some actors are given special rights which allow them to edit the ACL. Given the possibily of conflicts resulting from concurrent edits, Distributed ACLs are likely to rely on a Conflict-Free Replicated Data Type (CRDT) to encode changes to the list. 

## Design & Implementation

We've ended up with a generic decentralised group management system with fine-grained, per-member permissions. Once a group has been created, members can be added, removed, promoted and demoted. A group member can either be an individual (usually represented by a single public key) or another group. Assigned access levels can be restricted with application specific conditions.

### Access Levels

Each member has an associated access level which can be used to determine their permissions. The access levels we've defined are `Pull`, `Read`, `Write` and `Manage`. The precise access granted by each level is left open to interpretation but in the upcoming integration of our p2panda auth and encryption systems they will be as follows: `Pull` gives the ability to replicate encrypted data, `Read` gives the ability to decrypt data, `Write` gives the ability to mutate data and `Manage` gives the ability to mutate the group state.

Each access level is cumulative, meaning that it includes the rights granted by lower levels (ie. `Read` also includes `Pull` rights). Each access level can be assigned an associated set of conditions; this allows fine-grained partitioning of each access level. For example, `Read` conditions could be assigned with a path to restrict access to specific areas of a dataset. Finally, only members with `Manage` access are allowed to modify the group state by adding, removing, promoting or demoting other members.

### Group Control Operations

The aforementioned group actions are published as group control operations; each operation is cryptographically-signed, contains a group identifier and action and refers to `previous` operations and `dependencies`. Together, these operations form a causal Directed Acyclic Graph (DAG) which is used to track modifications to the group state over time. The `previous` field allows us to establish a causal "happened before" relationship between group actions, while the `dependencies` field offers a way to point to operations outside of the group which may need to be processed before the action is applied (such as application-specific dependencies or dependencies on other groups).

### Concurrency Resolver

Membership state for a group is maintained locally using a Causal-Length CRDT based on grow-only sets which allow for efficiently merging states across graph branches. However, it's simplicity does not allow us to fully handle conflicting group states emerging from some concurrent scenarios. In such cases, all operations in the DAG are walked in a depth-first search so that any "bubbles" of concurrent operations may be identified. Resolution rules are then applied to the operations in these bubbles in order to populate a filter of operations to be invalidated. Once the offending operations have been invalidated, any dependent operations are then invalidated in turn.

We have defined the `Resolver` as a Rust trait to allow for multiple implementations with contrasting rulesets for identifying which concurrent operations are to be considered invalid. This approaches arises from the understanding that applications have different requirements around trust and security; some may operate in high-stakes contexts where the most cautious implementation is always preferred, while others may operate in low-stakes contexts without the need for strict conflict resolution. The initial offering of our `p2panda-auth` crate offers a single resolver implementation which we refer to as a "strong removal" resolver. The ruleset is as follows:

1) Removal or demotion of a manager causes any concurrent actions by that member to be invalidated
2) Mutual removals, where two managers remove or demote one another concurrently, are not invalidated; both removals are applied to the group state but any other concurrent actions by those members are invalidated
3) Re-adds are allowed; if Alice removes Charlie then re-adds them, they are still a member of the group but all of their concurrent actions are invalidated
4) Invalidation of transitive operations; invalidation of an operation due to the application of the aforementioned rules results in all dependent operations being invalidated 

We fully realise, as mentioned before, that this ruleset is not optimal or desirable for all cases. For example, an alternative implementation of the `Resolver` might utilise the seniority of a member to act as a tie-breaker in the case of a mutual removal. In that scenario, the member who was added to the group first would remain in the group and the more recently added member would be removed.

Another scenario: what happens when the only manager of a group is removed or demoted? Is the group state forever frozen? Or are all group members automatically promoted to managers? The flexibility of our approach allows for both options to be catered for. We look forward to further discussions around these different requirement scenarios and would to assist anyone who wishes to implement their own custom resolver for `p2panda-auth`.

### Debugging Graphs

When trying to reason about various group membership and access control scenarios, it can become extremely challenging to hold complex operation graphs in one's head. We spent many hours dicussing such scenarios during the design of our system and often resorted to sketching diagrams to try and gain an understanding of what was happening. To make things easier, we've implemented a means of printing an auth group graph (using graphviz) to allow for visualising the group control message DAG. This is especially helpful when debugging group state and understanding the impact of concurrency.

![Transitive Invalidation Graph](/assets/images/250716-transitive-invalidation-graph.jpg)

> Here we have a visualisation of a DAG of group membership operations. The graph is read from bottom to top. A group is created with two initial members: individuals A and B. Both of these individuals are assigned manage-access. We then have two concurrent branches of operations. In the right-hand branch: individual A removes B. In the left-hand branch: individual B adds C to the group and assigns them manage-access (this operation is shown with a red background); C then adds individual D to the group with read-access (this operation is shown with an orange background). Since individual B is removed in the right-hand branch, any concurrent actions of theirs are invalidated (shown in red) and any dependent (aka.  transitive) actions are also invalidated (shown in orange).  A "Group Members" table with a green background shows individual A as the only member; this is a representation of the resolved state of the DAG.

### Nested Groups

Each member in a p2panda auth group can either be an individual or a group. Individuals are understood to be "stateless" due to the fact that they represent a single immutable identity. Groups, on the other hand, are understood to be "stateful" as they contain a mutable set of members. Defining group members in this way allows us to create nested group relationships, where a single group may contain several sub-groups as members.

![Nested Group Graph](/assets/images/250716-nested-group-graph.jpg)

> Here we have another visualisation of a DAG of group membership operations, this time illustrating a nested group scenario. A group 'T' is created with a single initial member: individual A with manage-access. Separately, a group D is created with two initial members: individuals L (manage-access) and M (write-access). Individual A adds individual B to group 'T' with manage-access. Individual B then adds individual C to group 'T' with read-access. The last operation in the graph points the creation of the 'D' group as a dependency and adds that group to group 'T' with manage-access. A "Group Members" table with a green background shows five members: A: manage, B: manage, C: read, L: manage and M: write.

## Challenges

### Centralised vs. Decentralised

Group management and access control is relatively straightforward in a centralised context where a server is the single source of truth and all group updates are received in total order. The server "knows" exactly which actions occurred before the others and is able to validate each one before
updating the group state or allowing access of a specific resource to a member. This means that there can never be conflicting group state (unless in the case of a bug or exploited vulnerability). We don't have such luxuries when building peer-to-peer systems.

In our world, a peer in the network may receive group updates in any order. To make matters worse, there may be long delays between when a group action is taken and when it's learned about by other peers in a network (this could even take years!). So, unlike centralised systems, we have to rely on partial order of actions (we know that one action happened after another but we don't know exactly when each one happened) to detect concurrent modifications of the group state and then apply specific rules to ensure that all members will eventually converge on the same state.

We also need to take into account malicious actors who may try to manipulate the group state for their own gains; for example, to harrass other members or retain access to their data. It is this paired need to ensure eventual consistency in the face of concurrency and negate byzantine actors that drives many of our design decisions.

### Complex Edge Cases

As we've already mentioned, concurrency brings about some complex and challenging scenarios for group management and eventual consistency. Here we outline a few such scenarios and describe how our current implementation handles them.

**Mutual Removal Involving Byzantine Actor**

Penguin is a group manager and promotes Parrot to manager access level. Right afterward, Penguin changes her mind about Parrot and immediately demotes him. Parrot quite enjoys his promotion to manager status and chooses to ignore Penguin's demotion action. As a result, all of Parrot's future actions are technically considered concurrent with Penguin's demotion action. Parrot then goes on to demote all other group members, making him then single authority figure in the group.

![Concurrent Mutual Removal](/assets/images/250716-concurrent-mutual-removal.png)

> Here we have a Lamport Clock depicting a case of "mutual removal" involving a byzantine actor, as described in the paragraph above.

In this scenario, a third group member (such as Duck) who has received Penguin's demotion of Parrot and Parrot's demotion of Penguin will determine that a concurrent, mutual removal has occurred. As such, they will remove both Penguin and Parrot from the group and roll-back or ignore any subsequent actions by those two members. This ensures that Parrot's nefarious plan is ultimately undone.

It's worth noting that in a group with only two managers, a mutual removal effectively freezes the group membership state; no manager remains to add, remove, promote or demote other members. An alternative resolver implementation might choose to promote all remaining members to manager level in that case. Alternatively, one could rely on a seniority principle - where a remaining member with the longest history of group membership would be declared a manager. We have chosen what we believe to be a more conservative approach, where the remaining group members would need to create an entirely new group to re-establish manager roles.

**Concurrent Removal**

Duck creates a group and promotes Penguin to manager access level. Penguin receives the promotion control message after syncing with Duck and then decides to promote Parrot to manager. Parrot goes on to promote some of his friends to manager access level. At this point, without yet knowing about Penguin's promotion of Parrot and Parrot's subsequent actions, Duck choses to demote Penguin so that she is no longer a manager.

![Concurrent Removal](/assets/images/250716-concurrent-removal.png)

> Here we have another Lamport Clock, this one depicting a case of "concurrent removal", as described in the paragraph above.

In this scenario, since Penguin's promotion of Parrot and Duck's demotion of Penguin happened concurrently, Penguin is no longer a manager (since the demotion takes precedence) and any downstream actions taken by Penguin are ignored. This means that Parrot and his friends are no longer managers of the group and any actions they took as managers are invalidated.

### Broadcast-Only Contexts

Another open question which emerged during our work is how to achieve access control in broadcast-based systems; this includes systems which rely exclusively on gossip-based replication strategies. In such cases, we can't control who will receive the data - just like a community radio station can't control who listens to their broadcast. As long as we have strong encryption in place, we can at least control who is able to make sense of the received data. We consider this an open problem and look forward to discuss possible solutions with other researchers.

## Related Work

How does our approach compare with other decentralised access control systems?

### localfirst/auth

The TypeScript library [localfirst/auth](https://github.com/local-first-web/auth) uses groups ("teams") to define access-control and encryption boundaries. In their own words:

> This library provides a Team class, which wraps the signature chain and encapsulates the team's members, devices, and roles. With this object, you can invite new members and manage their permissions.

Group membership is managed using the operation-based CRDT library [CRDX](https://github.com/local-first-web/auth/tree/main/packages/crdx). Roles can be dynamically added to a group; a member's access-level is inferred from the roles they are assigned. Members assigned the special "admin" role can perform actions which change the group membership (ie. add/remove members, create/assign/remove roles). New members are invited to the group using [Seitan Tokens](https://book.keybase.io/docs/teams/seitan).

Our approach is similar in how group state is managed using an operation-based CRDT. We were quite influenced by the way in which localfirst/auth allows custom approaches to conflict resolution. We differ in our use of nested-groups and access-levels with associated conditions (rather than roles) to describe a group member's capabilities. Another difference is that we do not use invitation tokens.

### Keyhive

The Ink & Switch project [Keyhive](https://www.inkandswitch.com/keyhive/notebook/) also uses a groups abstraction for their integration of access-control and encryption systems into the [automerge](https://automerge.org/) CRDT library. They describe their approach as using "convergent capabilities", which are intended to be similar to object-capabilities while being partition-tolerant and therefore suitable for use with local-first/offline-first applications and CRDTs in general. Group membership is derived from delegation chains; any member can delegate the capability they hold to another actor. A previously-delegated capability can be revoked by the original delegator or any member with special "manage" authority.

Our approach uses similar access-levels with attached conditions and also makes uses of nested-groups. We differ in the fact that we only allow "manager" members to add new members to a group (rather than any member being able to delegate their own capability). That said, it's still possible for users to follow a similar delegation approach using nested groups with the [Principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) (POLA).

Both localfirst/auth and Keyhive use something similar to a [Cryptree](https://ieeexplore.ieee.org/document/4032481) for data encryption. This is different from our approach which can be read about in detail [here](https://p2panda.org/2025/02/24/group-encryption.html).

## What's next?

So far most of the work has gone into how access levels are defined and associated with individuals or groups of actors. The next steps are integrating this system with `p2panda-encryption` and to take on the task of how we associate groups with a set of application data. This final piece of the puzzle is tentatively named `p2panda-spaces`.
