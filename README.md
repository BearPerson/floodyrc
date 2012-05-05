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
downtime, but I have a full-time job that will eat most of my energy,
and it would be irresponsible of me
to spend so much outside that I'm less effective at my job.


Missing things
--------------

Aka TODO:

* all protocol details
* 1-to-1 messaging
* routing details
* uprouting to an authority
