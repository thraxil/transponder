# Transponder

Still figuring out how to explain what this is.

It's kind of an architecture and a set of tools and components for
building applications in the architecture.

## Overview

The basic idea is kind of a data bus architecture. It makes heavy use
of ZeroMQ (0MQ) and aims to eliminate coupling between subsystems,
allowing you to build truly reusable components that can be composed
into isolated applications.

The heart of the system is the XPDR (pronounced "transponder"), which
listens on one set of 0MQ ports and re-broadcasts on another
set. Other components of the system only need to know how to connect
to the XPDR; none of them need to know how to communicate with each
other (or even that the others exist). 0MQ is supports pretty much
every programming language you might want to work with, so different
pieces of your system can be written in different languages, allowing
you to make use of whatever libraries may already exist.

A typical system will have the XPDR, one or more "frontend" nodes that
take HTTP requests coming in and send them into the XPDR as 0MQ
messages, then listens on the XPDR's output for a response, which it
sends back to the browser. Meanwhile one or more "worker" nodes listen
to the XPDR output for requests that they know how to process; when
one comes in, they do the work, and send a response back to the XPDR
(which will make it back to the original frontend).

The main use-case driving this design is web applications, but it
could actually be used for building any kind of networked application,
just requiring the right front-end protocol adaptor.

## XPDR

The XPDR is split into a control plane and a data plane (remember,
kids, in-band signalling is the devil). Most stuff will be going
through the data plane. The data plane has a REP socket for input and
a PUB socket for output. Anything coming into the REP simply gets
blindly sent out the PUB. The control plane is structured the same way
with a REP and PUB pair. The catch is that the XPDR also knows how to
interpret some messages that come in on the control plane and deal
with them. Eg, a 'pause' message will tell it to stop sending data
plane messages out until a 'resume' message comes in. This is useful
for seemlessly replacing or upgrading a worker without visible
downtime or dropped requests. The control plane will also get and pass
along messages for applying backpressure, heartbeats, and about
workers joining or leaving, which allows for good monitoring and
dynamic scaling (a "supervisor" worker could spawn more workers of a
particular type when throughput is dropping and then shut them back
down when traffic dies down).

## Frontend

Starting to look at the rest of the system to see how it fits
together, we'll start with a frontend node. A typical one for web apps
will be a process that listens on port 80 for HTTP requests. When it
gets a request, it generates a unique ID for that request, then
constructs a 0MQ message that contains the full HTTP request, the
frontend node's unique id, the message id, and makes a routing string,
probably from the HTTP method and path. Typically, a frontend would be
written in a language/framework allowing for cheap concurrency to
support very large numbers of simultaneous requests (eg, Go, node.js,
Erlang, gevent, tornado). The worker holds the socket to the browser
open, makes a mapping of the message id to that socket (so it can find
it later), then sends the message off to the XPDR. At the same time,
the frontend is subscribed to the XPDR's output and looking for
messages that are a reply to one of the messages it has sent out. When
it gets one, it tracks down the open socket, pulls the response out,
and ships that down to the browser, closing the socket if appropriate.

## Typical Application Worker

On the back end, one or more worker nodes are listening to the output
of the XPDR, looking for messages with a routing key that indicates
that they know how to handle it. Application workers might be much
heavier weight and more batch oriented. They could be single threaded,
taking one message at a time off their input queue, processing it,
sending a response back to the XPDR, and then moving on to the next.

## Other Components

Everything else we'll look at (and arguably, even the frontends) are
just variations on the workers. Let's discuss a few to get a better
sense of how this all fits together.

### Logger

This one could simply read every message that the XPDR broadcasts and
write an entry to a log file, or to syslog, never needing to send any
responses to anything. This is the advantage of the fanout PUB-SUB
nature of the XPDR output.

### Authenticator

This picks up all messages that are not marked as authenticated (and
that ought to be). It could look at the HTTP headers/cookies for a
session token, validate it, and send a message back out that's the
original message with the additional 'authenticated' attribute set and
username. Meanwhile, the normal application workers would be
configured to simply ignore any messages not marked as authenticated.

### Static Server

Looks for requests with the right path, maps them to a directory on
the filesystem, and sends them out.

### Proxy/Adaptor

Say you have a legacy Rails or PHP app and want to integrate it. A
proxy component would listen for incoming messages, convert them back
to an HTTP request, rewrite some location/path fields as needed, make
that request to the legacy app, then package its response up as a
message to go back to the XPDR.

### Replayer

Store the last N messages that have gone by (or all the messages for
the last N seconds) in a circular buffer. Watch for a "please replay"
message asking it to resend a particular message or messages from a
time period.

### Bridge

Take messages matching a certain pattern on one XPDR and pass them
along to another XPDR. For composing systems of systems.

### Stats Collector

Like statsd, count messages that go through and aggregate statistics
about them, rolling up every N seconds and submitting to Graphite (or
sending the aggregates as a new message).

### Console

Prints messages matching a particular pattern to the console. Allows
you to send commands in to the control plane from STDIN. Eg, you could
attach a console component to your running system, watching all the
messages go by in real-time. Then you send a 'pause' message which
causes it to stop moving messages along, watch until all the workers
have completed the job they were in the middle of handling, upgrade or
replace some of the workers, then resume the system and watch the
queues drain.

### Supervisor

Watches for heartbeat messages from a particular set of workers. Knows
how to start a replacement if one disappears.

### Scaler

Watch for messages about load, throughput, etc. and spin up/down new
workers in response.

## Message Format

per 0MQ norms, every message gets a routing key, which is a prefix string
that workers use to quickly decide whether they are going to handle
that message or not. After that, there are a number of header fields
and the payload of the message will typically be a JSON structure.

The routing key is fairly arbitrary but might look like:

* "HTTP|GET|host.example.com|unauthenticated|/path/requested/"
* "HTTP|GET|host.example.com|unauthenticated|/path/requested/"
* "RESPONSE|id=5f25e981-272d-4552-a433-22cb4de31272"
* "PAUSE|*"
* "RESUME|frontend.*"
* "HEARTBEAT|frontend|5f25e981-272d-4552-a433-22cb4de31272"

Technically, it's up to your system. But the way 0MQ subscriptions
work, you'll want to set it up so workers can subscribe to
prefixes. Generally, there will be a lot of messages flying around in
a moderately complex system, so the goal is to make it as simple as
possible for a worker to know that it can ignore a given message.

## Deployment/Orchestration
