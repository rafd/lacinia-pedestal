Subscriptions
=============

Subscriptions are a way for a client to request notifications about arbitrary events defined by the server;
this parallels how a query exposes arbitrary data defined by the server.

The essential support for GraphQL subscriptions is in the
`main Lacinia library <http://lacinia.readthedocs.io/en/latest/subscriptions/index.html>`_.

Lacinia-Pedestal's subscription support is designed to be compatible with
`Apollo GraphQL <https://github.com/apollographql/subscriptions-transport-ws>`_, a popular library
in the JavaScript domain [#apollo]_.
Like Apollo, Lacinia-Pedestal uses `WebSockets <https://en.wikipedia.org/wiki/WebSocket>`_ to create a durable connection between the client and the server.

Overview
--------

A client (typically, a web browser or mobile phone) will establish a connection to the server,
and convert it to a full-duplex WebSocket connection.

This single WebSocket connection will be multiplexed to handle any number of subscription requests
from the client.

When a subscription is requested, a `streamer <http://lacinia.readthedocs.io/en/latest/subscriptions/streamer.html>`_
defined in the GraphQL schema is invoked.
A *streamer* is similar to a field resolver; it has two responsibilities:

* Do whatever setup is necessary, then as new events are available,
  provide data to a source stream callback function.

* Return a cleanup function that shuts down whatever was previously set up.

Most commonly, a streamer will subscribe to some external feed such as a JMS or Kafka queue, or perhaps
a `core.async pub <http://clojure.github.io/core.async/#clojure.core.async/pub>`_ or channel.

When a streamer passes nil to the callback, a clean shutdown of the subscription occurs; the
client is sent a completion message.
The completion message informs the client that the stream of events has completed, and that it
should not attempt to reconnect.

The definition of "completed" here is entirely up to the application.
For example, a field argument could specify the maximum number of values to stream, and the
streamer can pass nil after sufficient values are streamed.

The cleanup function is invoked when the client closes the subscription, when the connection from
the client is lost due to a network partition, or when the streamer passes nil to the callback.

Configuration
-------------

When using ``com.walmartlabs.lacinia.pedestal2/default-service``, subscriptions are always
enabled, but ``default-service`` is always intended to be replaced in a live application.

The underlying function ``com.walmartlabs.lacinia.pedestal2/enable-subscriptions`` does
the work of enabling subscriptions; the function is passed subscription options:


The following keys are commonly used:

.. glossary::

  ``:subscriptions-path``
    Path to use in subscriptions WebSocket requests; defaults to ``/ws``,

  ``:keep-alive-ms``
    The interval at which keep-alive messages are sent to the client; defaults to 30 seconds.

  ``:subscription-interceptors``
    A seq of interceptors used when processing GraphQL query, mutation, or subscription requests
    via the WebSocket connection. This is used when overriding the default interceptors.

Further options are described by `listener-fn-factory <https://walmartlabs.github.io/apidocs/lacinia-pedestal/com.walmartlabs.lacinia.pedestal.subscriptions.html#var-listener-fn-factory>`_.

Connection Parameters
---------------------

When the client creates a connection, it may pass a payload in the ``connection_init`` message;
this is the connection parameters, and is made available to the streamer and resolver in
the context under the ``:com.walmartlabs.lacinia/connection-parameters`` key.

Endpoint
--------

Subscriptions are processed on a second endpoint; normal requests continue to be sent to ``/api``, but
subscription requests must use ``/ws``.

The ``/ws`` endpoint does not handle ordinary requests; instead it is used only to establish the
WebSocket connection.
From there, the client sends WebSocket text messages to initiate a subscription, and
the server sends WebSocket text messages for subscription updates and keep alive messages.

Subscription requests are not allowed in ``/api`` path.

GraphiQL
--------

GraphiQL, when enabled, is configured with subscriptions enabled; this means that GraphiQL can send ``subscription`` queries.


.. [#apollo] Apollo defines a `particular contract <https://github.com/apollographql/subscriptions-transport-ws/blob/master/PROTOCOL.md>`_
  for how the client and server communicate; this includes heartbeats, and an explicit way for
  the server to signal to the client that the subscription has completed.

  The Apollo project also provides `clients in several languages <https://github.com/apollographql>`_.
