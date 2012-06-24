txredisapi
==========

*For the latest source code, see <http://github.com/fiorix/txredisapi>.*


``txredisapi`` is a non-blocking client driver for the [redis](http://redis.io)
database, written in Python. It uses [Twisted](http://twistedmatrix.com) for
the asynchronous communication with redis.

It started as a fork of the original
[redis protocol for twisted](http://pypi.python.org/pypi/txredis/), and evolved
into a more robust, reliable, and complete solution for applications like web
servers. These types of applications often need a fault-tolerant pool of
connections with multiple redis servers, making it possible to easily develop
and maintain distributed systems.

Most of the [redis commands](http://redis.io/commands) are supported, as well
as other features such as silent reconnection, connection pools, and automatic
sharding.

This driver is distributed as part of the [cyclone](http://cyclone.io) web
framework.

### Features ###

- Connection Pools
- Lazy Connections
- Automatic Sharding
- Automatic Reconnection
- Publish/Subscribe (PubSub)
- Transactions
- Unix Socket Connections


Install
-------

Bear in mind that ``txredisapi.py`` is pure-python, in a single file.
Thus, there's absolutely no need to install it. Instead, just copy it to your
project directory and start using.

Latest source code is at <https://github.com/fiorix/txredisapi>.

If you have [cyclone](http://cyclone.io), you probably already have it too.
Try the following:

    $ python
    >>> import cyclone.redis
    >>> cyclone.redis.version
    '0.4'

However, if you really really insist in installing, get it from pypi:

    pip install txredisapi


### Unit Tests ###

[Twisted Trial](http://twistedmatrix.com/trac/wiki/TwistedTrial) unit tests
are available. Just start redis, and run ``trial ./tests``.
If *unix sockets* are disabled in redis, it will silently skip those tests.

There are live test results at <https://secure.travis-ci.org/fiorix/txredisapi>.

![Build Status](https://secure.travis-ci.org/fiorix/txredisapi.png)


Usage
-----

First thing to do is choose what type of connection you want. The driver
supports single connection, connection pools, sharded connections (with
automatic distribution based on a built-in consistent hashing algorithm),
sharded connection pools, and all of these different types can be *lazy*,
which is explained later (because I'm lazy now).

Basically, you want normal connections for simple batch clients that connect
to redis, execute a couple of commands and disconnect - like crawlers, etc.

Example:

	#!/usr/bin/env python
	# coding: utf-8

	import txredisapi as redis

	from twisted.internet import defer
	from twisted.internet import reactor


	@defer.inlineCallbacks
	def main():
	    rc = yield redis.Connection()
	    print rc

	    yield rc.set("foo", "bar")
	    v = yield rc.get("foo")
	    print "foo:", repr(v)

	    yield rc.disconnect()


	if __name__ == "__main__":
	    main().addCallback(lambda ign: reactor.stop())
	    reactor.run()


Easily switch between ``redis.Connection()`` and ``redis.ConnectionPool()``
with absolutely no changes to the logic of your program.

These are all the supported methods for connecting to Redis::

	Connection(host, port, dbid, reconnect)
	lazyConnection(host, port, dbid, reconnect)

	ConnectionPool(host, port, dbid, poolsize, reconnect)
	lazyConnectionPool(host, port, dbid, poolsize, reconnect)

	ShardedConnection(hosts, dbid, reconnect)
	lazyShardedConnection(hosts, dbid, reconnect)

	ShardedConnectionPool(hosts, dbid, poolsize, reconnect)
	lazyShardedConnectionPool(hosts, dbid, poolsize, reconnect)

	UnixConnection(path, dbid, reconnect)
	lazyUnixConnection(path, dbid, reconnect)

	UnixConnectionPool(unix, dbid, poolsize, reconnect)
	lazyUnixConnectionPool(unix, dbid, poolsize, reconnect)

	ShardedUnixConnection(paths, dbid, reconnect)
	lazyShardedUnixConnection(paths, dbid, reconnect)

	ShardedUnixConnectionPool(paths, dbid, poolsize, reconnect)
	lazyShardedUnixConnectionPool(paths, dbid, poolsize, reconnect)


The arguments are:

- host: the IP address or hostname of the redis server. [default: localhost]
- port: port number of the redis server. [default: 6379]
- path: path of redis server's socket [default: /tmp/redis.sock]
- dbid: database id of redis server. [default: 0]
- poolsize: how many connections to make. [default: 10]
- reconnect: auto-reconnect if connection is lost. [default: True]
- hosts (for sharded): list of ``host:port`` pairs. [default: None]
- paths (for sharded): list of ``pathnames``. [default: None]


### Connection Handlers ###

All connection methods return a connection handler object at some point.

Normal connections (not lazy) return a deferred, which is fired with the
connection handler after the connection is established.

In case of connection pools, it will only fire the callback after all
connections are set up, and ready.

Connection handler is the client interface with redis. It accepts all the
commands supported by redis, such as ``get``, ``set``, etc. It is the ``rc``
object in the example below.

Connection handlers will automatically select one of the available connections
in connection pools, and automatically reconnect to redis when necessary.

If the connection with redis is lost, all commands will raise the
``ConnectionError`` exception, to indicate that there's no active connection.
However, if the ``reconnect`` argument was set to ``True`` during the
initialization, it will continuosly try to reconnect, in background.

Example:

	#!/usr/bin/env python
	# coding: utf-8

	import txredisapi as redis

	from twisted.internet import defer
	from twisted.internet import reactor


	def sleep(n):
	    d = defer.Deferred()
	    reactor.callLater(5, lambda *ign: d.callback(None))
	    return d


	@defer.inlineCallbacks
	def main():
	    rc = yield redis.ConnectionPool()
	    print rc

	    # set
	    yield rc.set("foo", "bar")

	    # sleep, so you can kill redis
	    print "sleeping for 5s, kill redis now..."
	    yield sleep(5)

	    try:
		v = yield rc.get("foo")
		print "foo:", v

		yield rc.disconnect()
	    except redis.ConnectionError, e:
		print str(e)


	if __name__ == "__main__":
	    main().addCallback(lambda ign: reactor.stop())
	    reactor.run()


### Lazy Connections ###

This type of connection will immediately return the connection handler object,
even before the connection is made.

It will start the connection, (or connections, in case of connection pools) in
background, and automatically reconnect if necessary.

You want lazy connections when you're writing servers, like web servers, or
any other type of server that should not wait for the redis connection during
the initialization of the program.

The example below is a web application, which will expose redis set, get and
delete commands over HTTP.

If the database connection is down (either because redis is not running, or
whatever reason), the web application will start normally. If connection is
lost during the operation, nothing will change.

When there's no connection, all commands will fail, therefore the web
application will respond with HTTP 503 (Service Unavailable). It will resume to
normal once the connection with redis is re-established.

Try killing redis server after the application is running, and make a couple
of requests. Then, start redis again and give it another try.

Example:

	#!/usr/bin/env python
	# coding: utf-8

	import sys

	import cyclone.web
	import cyclone.redis
	from twisted.internet import defer
	from twisted.internet import reactor
	from twisted.python import log


	class Application(cyclone.web.Application):
	    def __init__(self):
		handlers = [ (r"/text/(.+)", TextHandler) ]

		RedisMixin.setup()
		cyclone.web.Application.__init__(self, handlers, debug=True)


	class RedisMixin(object):
	    redis_conn = None

	    @classmethod
	    def setup(self):
            RedisMixin.redis_conn = cyclone.redis.lazyConnectionPool()


	# Provide GET, SET and DELETE redis operations via HTTP
	class TextHandler(cyclone.web.RequestHandler, RedisMixin):
	    @defer.inlineCallbacks
	    def get(self, key):
		try:
		    value = yield self.redis_conn.get(key)
		except Exception, e:
		    log.msg("Redis failed to get('%s'): %s" % (key, str(e)))
		    raise cyclone.web.HTTPError(503)

		self.set_header("Content-Type", "text/plain")
		self.write("%s=%s\r\n" % (key, value))

	    @defer.inlineCallbacks
	    def post(self, key):
            value = self.get_argument("value")
            try:
                yield self.redis_conn.set(key, value)
            except Exception, e:
                log.msg("Redis failed to set('%s', '%s'): %s" % (key, value, str(e)))
                raise cyclone.web.HTTPError(503)

            self.set_header("Content-Type", "text/plain")
            self.write("%s=%s\r\n" % (key, value))

	    @defer.inlineCallbacks
	    def delete(self, key):
            try:
                n = yield self.redis_conn.delete(key)
            except Exception, e:
                log.msg("Redis failed to del('%s'): %s" % (key, str(e)))
                raise cyclone.web.HTTPError(503)

            self.set_header("Content-Type", "text/plain")
            self.write("DEL %s=%d\r\n" % (key, n))


	def main():
	    log.startLogging(sys.stdout)
	    reactor.listenTCP(8888, Application(), interface="127.0.0.1")
	    reactor.run()


	if __name__ == "__main__":
	    main()


This is the server running in one terminal::

	$ ./helloworld.py
	2012-02-17 15:40:25-0500 [-] Log opened.
	2012-02-17 15:40:25-0500 [-] Starting factory <redis.Factory instance at 0x1012f0560>
	2012-02-17 15:40:25-0500 [-] __main__.Application starting on 8888
	2012-02-17 15:40:25-0500 [-] Starting factory <__main__.Application instance at 0x100f42290>
	2012-02-17 15:40:53-0500 [RedisProtocol,client] 200 POST /text/foo (127.0.0.1) 1.20ms
	2012-02-17 15:41:01-0500 [RedisProtocol,client] 200 GET /text/foo (127.0.0.1) 0.97ms
	2012-02-17 15:41:09-0500 [RedisProtocol,client] 200 DELETE /text/foo (127.0.0.1) 0.65ms
	(killed redis-server)
	2012-02-17 15:48:48-0500 [HTTPConnection,0,127.0.0.1] Redis failed to get('foo'): Not connected
	2012-02-17 15:48:48-0500 [HTTPConnection,0,127.0.0.1] 503 GET /text/foo (127.0.0.1) 2.99ms


And these are the requests, from ``curl`` in another terminal.

Set:

	$ curl -D - -d "value=bar" http://localhost:8888/text/foo
	HTTP/1.1 200 OK
	Content-Length: 9
	Content-Type: text/plain

	foo=bar

Get:

	$ curl -D - http://localhost:8888/text/foo
	HTTP/1.1 200 OK
	Content-Length: 9
	Etag: "b63729aa7fa0e438eed735880951dcc21d733676"
	Content-Type: text/plain

	foo=bar

Delete:

	$ curl -D - -X DELETE http://localhost:8888/text/foo
	HTTP/1.1 200 OK
	Content-Length: 11
	Content-Type: text/plain

	DEL foo=1

When redis is not running:

	$ curl -D - http://localhost:8888/text/foo
	HTTP/1.1 503 Service Unavailable
	Content-Length: 89
	Content-Type: text/html; charset=UTF-8

	<html><title>503: Service Unavailable</title>
	<body>503: Service Unavailable</body></html>


### Sharded Connections ###

They can be normal, or lazy connections. They can be sharded connection pools.
Not all commands are supported on sharded connections.

If the command you're trying to run is not supported on sharded connections,
the connection handler will raise the ``NotImplementedError`` exception.

Simple example with automatic sharding of keys between two redis servers:

	#!/usr/bin/env python
	# coding: utf-8

	import txredisapi as redis

	from twisted.internet import defer
	from twisted.internet import reactor


	@defer.inlineCallbacks
	def main():
	    rc = yield redis.ShardedConnection(["localhost:6379", "localhost:6380"])
	    print rc
	    print "Supported methods on sharded connections:", rc.ShardedMethods

	    keys = []
	    for x in xrange(100):
		key = "foo%02d" % x
		yield rc.set(key, "bar%02d" % x)
		keys.append(key)

	    # yey! mget is supported!
	    response = yield rc.mget(keys)
	    for val in response:
		print val

	    yield rc.disconnect()


	if __name__ == "__main__":
	    main().addCallback(lambda ign: reactor.stop())
	    reactor.run()


### Transactions ###

For obvious reasons, transactions are NOT supported on sharded connections.
But they work pretty good on normal or lazy connections, and connection pools.

NOTE: redis uses the following methods for transactions:

- MULTI: start the transaction
- EXEC: commit the transaction
- DISCARD: you got it.

Because ``exec`` is a reserved word in Python, the command to commit is
``commit``.

Example:

	#!/usr/bin/env python
	# coding: utf-8

	import txredisapi as redis

	from twisted.internet import defer
	from twisted.internet import reactor


	@defer.inlineCallbacks
	def main():
	    rc = yield redis.ConnectionPool()

	    # Remove the keys
	    yield rc.delete(["a1", "a2", "a3"])

	    # Start transaction
	    t = yield rc.multi()

	    # These will return "QUEUED" - even t.get(key)
	    yield t.set("a1", "1")
	    yield t.set("a2", "2")
	    yield t.set("a3", "3")
	    yield t.get("a1")

	    # Try to call get() while in a transaction.
	    # It will fail if it's not a connection pool, or if all connections
	    # in the pool are in a transaction.
	    # Note that it's rc.get(), not the transaction object t.get().
	    try:
		v = yield rc.get("foo")
		print "foo=", v
	    except Exception, e:
		print "can't get foo:", e

	    # Commit, and get all responses from transaction.
	    r = yield t.commit()
	    print "commit=", repr(r)

	    yield rc.disconnect()


	if __name__ == "__main__":
	    main().addCallback(lambda ign: reactor.stop())
	    reactor.run()


Calling ``commit`` will cause it to return a list with the return of all
commands executed in the transaction. ``discard``, on the other hand, will
normally return just an ``OK``.


### Authentication ###

This is how to authenticate::

	#!/usr/bin/env python

	import txredisapi
	from twisted.internet import defer
	from twisted.internet import reactor


	@defer.inlineCallbacks
	def main():
	    redis = yield txredisapi.Connection()
	    yield redis.auth("foobared")
	    yield redis.set("foo", "bar")
	    print (yield redis.get("foo"))
	    reactor.stop()


	if __name__ == "__main__":
	    main()
	    reactor.run()

If the password does not match, most of the commands will return nothing,
except for ``get``, which returns ``operation not permitted``.

There's one caveat: whenever authentication is required, the *database id* must
be manually selected after the ``auth`` command. The ``dbid=N`` argument of
``Connection()`` must not be defined, or set to ``None``; otherwise, it'll try
to select *dbid* before authentication, and it will fail.


Credits
=======
Thanks to (in no particular order):

- Gleicon Moraes

  - Bug fixes, testing, and [RestMQ](http://github.com/gleicon/restmq>).
  - For writing the Consistent Hashing algorithm used for sharding.

- Dorian Raymer and Ludovico Magnocavallo

  - Authors of the original *redis protocol for twisted*.

- Vanderson Mota

  - Initial pypi setup, and patches.

- Jeethu Rao

  - Contributed with test cases, and other ideas like support for travis-ci
