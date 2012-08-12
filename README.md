floodyrc
========

Yet another IRC redesign.

Well, a redesign of the mass-messaging ideas that everyone knows from IRC.

Non-goals
---------

* Extending featureset
* full backwards compatibility

We will
-------

* increase bandwidth usage (expected factor ~2)
* break your assumptions
* be harder to implement than IRC (both server and client)
* not specify semantics (at first), focusing on underlying design.
* make (robust) clients more CPU/memory intensive

Goals
-----

* Be more robust against component/link failures than IRC
* Allow more secure models (pubkey crypto, anyone?)
* Build a robust foundation that you can use to implement your idea of "chat"

1-minute-intro
--------------

* Four moving parts:
  * Client program (that's you)
  * Routing system (you connect here, they send you messages)
  * Authority system (lets you join channels and ban people)
  * Name resolution (translates human channel names into machine addresses)
* Clients are expected to open multiple connections and deduplicate
* Routing system is not strongly ordered, dumb, and wastefully implemented.
  But pretty dang reliable.
* Authority system is what you used to call IRC services, but better.


15-minute-intro
===============

Channels
--------

Our idea of "channel" is fairly similar to IRC:

* Your client explicitly joins and then starts receiving messages
* You can probably send messages once you're joined, too
* Channels have admins that can ban people

Networks
--------

We break the traditional idea of "network".
Global naming is a bad idea.

* Namespaces are external. Anyone can run naming that points at stuff.
* The teams running the authority servers are roughly equivalent to "IRCops."
  "K-line" turns into "ban on all channels managed by my authority servers."
* Multiple authority servers can coexist on the same routing system.
  There has to be some trust relationship between the two, so you won't have
  completely open routing systems, but a little crypto goes a long way...

Clients
-------

Traditional IRC clients have a SPOF in their connection
to the server and the stability of that server.
To fix that, we push more logic into clients.
Specifically, we expect them to:

* Connect to the routing layer in multiple places for redundancy
* De-duplicate incoming messages (this is expensive)
* Retry requests after timeouts, with exponential backoff

These things suck to implement, but are needed for reliability.
People can always deploy client "gateways" that offer, say,
an IRC-compatible protocol to whoever wants one.

Routing
-------

We split out message routing.
Ideally, the routing layer is dumb, but reliable.
IRC aches from having all servers know all channels and all users.

Our routing nodes have significantly less state,
which should let us scale better if needed.
Deploy more routing nodes for more clients,
or if you really need local routers on every battleship.

Multicast routing algorithms are hard. Suggested approaches:

* Use a tree (efficient like IRC, fails like IRC)
* Make a graph with node order ~4, flood
  (doubles traffic, more reliable, harder to implement, much harder to maintain)
* Talk to a researcher in the field. Be sure to mention you need
  low-latency failover and would like reliableish messages

Authority
---------

The authority system is the fun part.
Here's where we tell you "go nuts, everything goes."
Confused yet? Here's what we expect:
* All join requests go to the channel's authority for confirmation
* Bans and op changes go to the authority
* Authorities know who is on the channel
* If authority is down, you can still talk, but joins/bans/ops will time out
* There will be many custom authority implementations whenever someone
  invents a new style of banning people

Authorities should be pretty consistent on the "who is op, who is banned" part.
We won't kill you if they are not, but your users might.
Authorities should probably make use of local disks, and have backups.
To implement consistency, you could:

* Only run a single authority server per channel
* Have multiple in a master-slave setup, with (manual?) failover
* Implement distributed locking
* Use a distributed message queue (paxos, anyone?)

Naming
------

Good naming is hard. For now, let's say

* User provides client with a channel name (say comms.myproject.org/support-de)
* name resolution provides client with host:ports of applicable routing servers,
  and a channel identifier telling them who you want to talk to.
* You end up on a channel about horse porn because someone hacked the naming servers.

Projects/people that want hard control over their namespace are expected
to run their own naming, which should be cheap (files served via HTTP could do).
To "regain" channel control, you could work with authority admins,
or create a new channel and repoint the name.

To avoid "malicious redirect" problems, channels might have a "known" list
of names pointing to them, with clients erroring out if the channel
has a different list.
However, that's already specifying more than I'm comfortable with here...

Channel models
--------------

I expect we will implement some, if not all, of the following:

* Fully open (anyone can join, anyone can message. No ops. Does not need an authority. The most reliable, except against DoS. No userlist.)
* Semi-open (anyone can join. Ops can mute, but not ban. While authority is down, new joins start muted. Existing users can still talk. Userlists unreliable.)
* Managed (Ops can mute and ban. All joins go through authority. While authority is down, you cannot join, but still talk if you are already joined.)
* Filtered (Ops can mute, ban, and setup message filters. All joins and all messages go through authority. While authority is down, channel is unusable.)

Security
--------

Intentionally left un[der]specified.
At the bare minimum, routing and authority trust each other,
clients identify with random IDs.
It should not be hard to implement pubkey security, though,
in which case everyone trusts the routing layer to not drop messages,
and otherwise trust relevant keys.


Other notes
===========

Code will be (mostly) C++ because I need more practice in the language.
If I manage to write a full prototype, I expect people will scrap
it and reimplement the ideas properly. They should, it's prototype code.

Wire messages will be google protobufs, because I like them.
If you disagree, you're free to write your prototype using SOAP.

License for the prototype will be BSDish, because I don't like getting
into arguments over license terms and fine print. I wrote the code because
I thought it interesting at the time, not to make the world a better place.

Progress will be slow. This is my pet project to ponder and code during
downtime, but I have a full-time job that will eat most of my energy.


Design scratchpad
=================

Expect ranty, badly-formatted notes below.


Routing routers
---------------

Routers should keep as little global state as possible.
For the routing layer, each message originated in exactly one router.
If endpoints send messages redundantly,
the copies will be treated as separate messages by the routers.
(Might want to optimize this later. Will do for now.)

Routing is fundamentally exchangeable: Protocol promises messages are probably delivered,
not how. Also, in the code, the actual routing engine should be cheaply exchangeable,
as it's essentially just the interface SetReceiver(Callback<Message>), Publish(Message).

The current plan is to go with an arbitrary network, and "flood" it.
Maybe sometime someone will tell me a better algorithm, but for now this should work.
It's a bit wasteful, but extremely reliable and will work around component failures instantly.

The basic idea:
* To publish a message, a router sends it out on all connected links.
* If a router receives a message for the first time, it forwards a copy on all links,
  except the one on which it received the message.
* If a router receives a message it saw before, it is dropped. (This is an expected case)
* For simplicity, all links are bidirectional. We could extend to directed links.

To implement this, we need a bit of global state:
Each router increments a local message ID every time it publishes (not forwards!) a message.
All routers need to know the "last seen" message ID of all other routers.
Thus, if you get a message from a router that is > your last-seen, forward, otherwise drop.

This is technically quadratic in the number of routers, but this state should be tiny
(The router ID, last-seen ID, timestamp),
so we could get above 1k routers before the overhead becomes noticeable.
(And if anyone complains, we can always run "Inner routers" that never publish messages,
and thus don't appear in tables)

This does "waste" bandwidth. To compute how much, let's look at a single message.
Note that every router (other than the publisher) has exactly one incoming link
on which it received the first copy of the message.
These links form a spanning tree of the network.
All other links had a copy forwarded through them from both sides.
(Otherwise they would be incoming links for one of the sides.)

A spanning tree over N nodes always has N-1 links.
(The links carrying messages once, in our model.)
Every extra link costs twice the bandwidth of an incoming link.
Thus, for a network with N nodes and K links,
we total (2K-N+1)/(N-1) as much bandwidth as the ideal solution.

Obviously, you should not run this protocol over a fully-linked network.
But take a square grid, for instance, where we get K=2N, for a linear cost factor of 3x.
Triangular grids cost 2x, hexagonal grids 5x.
Since chat bandwidth tends to be tiny,
this should be a small price to pay for superior reliability,
in a field that tends to charge in "orders of magnitude".
(In IRC, hub bandwidth tends to be tiny, except for the peaks when processing a netjoin...)

Advantages of the model:
* Since we pessimistically try all links, we instantly fail around broken or slow links/machines.
* For single sources, we preserve message ordering. (Even though the protocol doesn't require us to.)
* Relatively simple to implement
* Smallish memory overhead even for large networks
* Works on any connected network, no specific shape required
* No administrator action needed through non-catastrophic temporary outages
* As long as there is any path from a node to another, messages between them work

Disadvantages:
* No sophisticated algorithm, researchers won't take you seriously
* Significantly more traffic than an optimal solution
* Router disconnects are hard:
  Since each router only has local state and the global last-seen counters,
  routers do not know if another router left the network,
  they can only time out their map entry eventually.
* Working around long-term failures and judging which links to add for redundancy can be hard
* Implementing reconnects is hardish


Wire protocol
-------------

Protobufs. Which don't self-delimit, so we'll have to do something about that.
If we had SCTP, we could use that to define packets.
Would be all kinds of neat, and I want to implement it as a prototype,
but we'll need to support TCP as well to keep everyone happy.

Proposed format:
* 4-byte header/guard to ensure the last frame size was correct.
  Also used to indicate what type of link this is. Should disconnect as soon as this is wrong.
  Cheap proposal:
  * router-to-router: CR2R
  * router-to-client: CR2C
  * client-to-router: CC2R
  * router-to-authority: CR2A
  * authority-to-router: CA2R
* varint-encoded uint32 encoding the size of the following payload
* payload, as serialized protobuf (which message depends on link type)

For each link type, there will only ever be one message used as payload.
The message contains an enum field indicating its type,
and a bunch of optional sub-messages, one of which will be set.


Private messages without global naming
--------------------------------------

When talking on a channel, a client has a nick (enforced by router).
However, nothing inherently enforces that a nick is globally unique.
This reduces global state: Authorities need to assign nicks on join,
and should ensure uniqueness inside that channel, but do not need to
enforce global uniqueness (which, as anyone on IRC knows, is hard).

So how do we implement private messaging? Since you generally only
know a client exists by seeing them on a channel, you can always
message BearPerson@#defocus, to put it in IRC terms.
This should be routed to the authority, which then knows which
connections hook into the channel with that nick, and can forward
the message appropriately.

Of course, if people really want global naming, they are free to implement it.
I assume this will usually come with a pre-registration somewhere, to keep people
from spamming the nick namespace or trying to collide nicks.
I imagine this will be implemented as a pseudo-channel, that nobody can
(normally) talk on, only supports limited user listings, but implements
directed messages.

Yes, this way private messages will always go through an authority,
and as such, will be less reliable than channel messages.
If this becomes a problem, we should create ad-hoc channels for 1:1 communication.
I would rather not directly expose the internal routing IDs for client connections
to other clients for direct messages, to keep down abuse potential and to have
more flexibility in implementation. (Clients should be able to reconnect
arbitrarily around the network without causing visible issues.)


Tags and other Fancy Stuff
--------------------------

A somewhat popular concept in IRC redesigns is fancier "membership" models.
For instance, imagine a channel where unregistered users can only send messages
to ops, registered users can talk normally, and ops have a special ops-only
subchannel to talk on.

For a while, I thought we should implement this as a "tag" system -
channel messages would each have a tag attached, and the membership create
from authority would configure the routers with which tags this client was
supposed to receive and be able to send.

However, this isn't much different from having a separate channel for each tag.
Since channels only have a machine-parseable identifier, it's already up to
clients (and name resolution) how to represent them.
So with some redirection logic, you should be able to implement the above
scenario with a slightly fancy authority, assuming we tell clients to expect
that sort of thing.


Missing things
--------------

Aka TODO:

* all protocol details
* 1-to-1 messaging
* routing details
* uprouting to an authority
