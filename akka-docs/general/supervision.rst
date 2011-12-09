Supervision
===========

This chapter outlines the concept behind supervision, the primitives offered
and their semantics. For details on how that translates into real code, please
refer to the corresponding chapters for Scala and Java APIs.

What Supervision Means
----------------------

Supervision describes a dependency relationship between actors: the supervisor
delegates tasks to subordinates and therefore must respond to their failures.
When a subordinate detects a failure (i.e. throws an exception), it suspends
itself and all its subordinates and sends a message to its supervisor,
signaling failure.  Depending on the nature of the work to be supervised and
the nature of the failure, the supervisor has four basic choices:

#. Resume the subordinate, keeping its accumulated internal state
#. Restart the subordinate, clearing out its accumulated internal state
#. Terminate the subordinate permanently
#. Escalate the failure

It is important to always view an actor as part of a supervision hierarchy,
which explains the existence of the fourth choice (as a supervisor also is
subordinate to another supervisor higher up) and has implications on the first
three: resuming an actor resumes all its subordinates, restarting an actor
entails restarting all its subordinates, similarly stopping an actor will also
stop all its subordinates.

Each supervisor is configured with a function translating all possible failure
causes (i.e. exceptions) into one of the four choices given above; notably,
this function does not take the failed actor’s identity as an input. It is
quite easy to come up with examples of structures where this might not seem
flexible enough, e.g. wishing for different strategies to be applied to
different subordinates. At this point it is vital to understand that
supervision is about forming a recursive fault handling structure. If you try
to do too much at one level, it will become hard to reason about, hence the
recommended way in this case is to add a level of supervision.

Akka implements a specific form called “parental supervision”. Actors can only
be created by other actors—where the top-level actor is provided by the
library—and each created actor is supervised by its parent. This restriction
makes the formation of actor supervision hierarchies explicit and encourages
sound design decisions. It should be noted that this also guarantees that
actors cannot be orphaned or attached to supervisors from the outside, which
might otherwise catch them unawares. In addition, this yields a natural and
clean shutdown procedure for (parts of) actor applications.

What Restarting Means
---------------------

When presented with an actor which failed while processing a certain message,
causes for the failure fall into three categories:

* Systematic (i.e. programming) error for the specific message received
* (Transient) failure of some external resource used during processing the message
* Corrupt internal state of the actor

Unless the failure is specifically recognizable, the third cause cannot be
ruled out, which leads to the conclusion that the internal state needs to be
cleared out. If the supervisor decides that its other children or itself is not
affected by the corruption—e.g. because of conscious application of the error
kernel pattern—it is therefore best to restart the child. This is carried out
by creating a new instance of the underlying :class:`Actor` class and replacing
the failed instance with the fresh one inside the child’s :class:`ActorRef`;
the ability to do this is one of the reasons for encapsulating actors within
special references. The new actor then resumes processing its mailbox, meaning
that the restart is not visible outside of the actor itself with the notable
exception that the message during which the failure occurred is not
re-processed.

Restarting an actor in this way recursively restarts all its children in the
same fashion, whereby all parent–child relationships are kept intact. If this
is not the right approach for certain sub-trees of the supervision hierarchy,
you should choose to stop the failed actor instead, which will terminate all
its children recursively, after which that part of the system may be recreated
from scratch. The second part of this action may be implemented using the
lifecycle monitoring described next or using lifecycle callbacks as described
in :class:`Actor`.

What Lifecycle Monitoring Means
-------------------------------

In contrast to the special relationship between parent and child described
above, each actor may monitor any other actor. Since actors emerge from
creation fully alive and restarts are not visible outside of the affected
supervisors, the only state change available for monitoring is the transition
from alive to dead. Monitoring is thus used to tie one actor to another so that
it may react to the other actor’s termination, in contrast to supervision which
reacts to failure.

Lifecycle monitoring is implemented using a :class:`Terminated` message to be
received by the behavior of the monitoring actor, where the default behavior is
to throw a special :class:`DeathPactException` if not otherwise handled. One
important property is that the message will be delivered irrespective of the
order in which the monitoring request and target’s termination occur, i.e. you
still get the message even if at the time of registration the target is already
dead.

Monitoring is particularly useful if a supervisor cannot simply restart its
children and has to stop them, e.g. in case of errors during actor
initialization. In that case it should monitor those children and re-create
them or schedule itself to retry this at a later time.

Another common use case is that an actor needs to fail in the absence of an
external resource, which may also be one of its own children. If a third party
terminates a child by way of the ``stop()`` method or sending a
:class:`PoisonPill`, the supervisor might well be affected.
