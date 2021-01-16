Glokka = Global + Akka

Glokka is a Scala library that allows you to register and lookup actors by names
in an Akka cluster. See:

* `Erlang's "global" module <http://erlang.org/doc/man/global.html>`_
* `Akka's cluster feature <https://doc.akka.io/docs/akka/current/cluster-usage.html>`_

Glokka is used in `Xitrum <http://xitrum-framework.github.io/>`_ to implement
its distributed `SockJS <https://github.com/sockjs/sockjs-client>`_ feature.

See `Glokka's Scaladoc <http://xitrum-framework.github.io/glokka>`_.

SBT
---

::

  libraryDependencies += "tv.cntt" %% "glokka" % "2.6.1"

Create registry
---------------

::

  import akka.actor.ActorSystem
  import glokka.Registry

  val system    = ActorSystem("MyClusterSystem")
  val proxyName = "my proxy name"
  val registry  = Registry.start(system, proxyName)

* You can start multiple registry actors. They must have different ``proxyName``.
* For convenience, ``proxyName`` can be any String, you don't have to URI-escape it.

Register actor by props
-----------------------

::

  // For convenience, ``actorName`` can be any String, you don't have to URI-escape it.
  val actorName = "my actor name"

  // Props to create the actor you want to register.
  val props = ...

  registry ! Registry.Register(actorName, props)

If the named actor exists, the registry will just return it. You will receive:

::

  Registry.Found(actorName, actorRef)

Otherwise ``props`` will be used to create the actor locally (when the actor
dies, it will be unregistered automatically). You will receive:

::

  Registry.Created(actorName, actorRef)

If you don't need to differentiate ``Found`` and ``Created``:

::

  registry ! Registry.Register(actorName, props)
  context.become {
    case msg: Registry.FoundOrCreated =>
      val actorName = msg.name
      val actorRef  = msg.ref
  }

Register actor by ref
---------------------

::

  registry ! Registry.Register(actorName, actorRefToRegister)

If the actor has not been registered, or has already been registered with the
same name, you will receive:

::

  Registry.Registered(actorName, actorRef)

Otherwise if there's another actor that has been registered with the name, you
will receive:

::

  Registry.Conflict(actorName, otherActorRef, actorRefToRegister)

In this case, you may need to stop ``actorRefToRegister``, depending on your
application logic.

Lookup actor by name
--------------------

Send:

::

  registry ! Registry.Lookup(actorName)

You will receive:

::

  Registry.Found(actorName, actorRef)

Or:

::

  Registry.NotFound(actorName)

Tell
----

If you don't want to lookup and keep the actor reference:

::

  registry ! Registry.Tell(actorName, msg)

::

  registry ! Registry.Tell(actorName, props, msg)

* If the named actor exists, msg will be sent to it.
* Otherwise, `props` will be used to create the named actor, and msg will be sent to it.

Cluster
-------

Glokka can run in Akka non-cluster mode (local or remote). While developing, you
can run Akka in local mode, then later config Akka to run in cluster mode.

In cluster mode, Glokka uses
`Akka's Cluster Singleton Pattern <https://doc.akka.io/docs/akka/current/cluster-singleton.html>`_
to maintain an actor that stores the name -> actorRef lookup table.

Akka config file for a node should look like ``config_example/application.conf``
(note ``MyClusterSystem`` in the source code example above and in the config file).
