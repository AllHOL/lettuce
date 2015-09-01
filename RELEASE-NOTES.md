lettuce 4.0.Beta1 RELEASE NOTES

This release is a major release that introduces numerous changes like stateful connections,
the reactive API and many more. Lettuce 4.0 includes all features from lettuce 3.3.

This release contains some breaking changes. You may want to consult the
wiki at https://github.com/mp911de/lettuce/wiki/Migration-from-3.x-to-4.x
to check the migration guide.

All `Redis...Connection` and `Redis...AsyncConnection` interfaces are deprecated and replaced by new `...Commands`
interfaces.

The cluster API was extended to run a command on multiple nodes and invocation
to multi-key commands DEL, MGET, MSET and MSETNX perform automatic pipelining
if the keys belong to different slots/masters.

A couple of changes are breaking changes, and you need most likely to adopt
your code to use lettuce 4.0. lettuce 4.0 dropped Java 6 and 7 support and
requires Java 8 to run.

lettuce 4.0 needs Java 8 and cannot be used with Java 6 or 7.

If you need any support, meet lettuce at:

* Google Group: https://groups.google.com/d/forum/lettuce-redis-client-users
* Gitter: https://gitter.im/mp911de/lettuce
* Github Issues: https://github.com/mp911de/lettuce/issues


Reactive API
------------
Lettuce provides since its genesis two API's: Sync and async.
Now comes a third one: Reactive.
The reactive API uses rx-java and every command on the reactive API returns an Observable.

```java
StatefulRedisConnection<String, String> connection = client.connect();
RedisReactiveCommands<String, String> reactive = stateful.reactive();

Observable<String> observable = reactiveApi.keys("*");
observable.subscribe();

// more sophisticated

reactive.keys("*").flatMap(reactive::del).subscribe();
```

The command is only executed on a subscription to the Observable.
Commands returning a collection, such as Lists or Sets,
return Observables that provide the element type.
The reactive API covers the same commands as the synchronous/asynchronous API's.

Read more: https://github.com/mp911de/lettuce/wiki/Reactive-API-(4.0)

Rearchitecting the API
-----
Before 4.0, connection resources (sockets, events) were bound to the particular API.
If one wanted to use a different API, he had to create another connection to Redis.
This coupling is loosed now. By calling `connect()` you will no longer get a synchronous connection, you will get a `StatefulRedisConnection`
with the access to the synchronous, asynchronous and reactive API.
All commands are executed using netty and it does not matter, from which API you come.

3.x code:
```java
RedisConnection connection = client.connect();
```

4.x code:
```java
StatefulRedisConnection stateful = client.connect();

RedisCommands commands = stateful.sync();

// still working
RedisConnection connection = stateful.sync();
```

The other `connect` methods like `connectAsync` and `connectSentinelAsync` are
deprecated but remain unchanged.

Breaking changes in `connect` methods are:

* `RedisClient.connect` (provides a `StatefulRedisConnection`)
* `RedisClient.connectPubSub` (provides a `StatefulRedisPubSubConnection`)
* `RedisClusterClient.connectCluster` (provides a `StatefulRedisClusterConnection`)

New `connect` methods:

* `RedisClient.connectSentinel` (provides a `StatefulRedisSentinelConnection`)

New segregated command interfaces
---------------------------------
Starting with reactive API's, another 13 interfaces will be provided with lettuce.
This change increases the count of types within the `com.lambdaworks.redis`
package and the package gets messier again. In combination with the stateful connection
the original `...Connection` or `...AsyncConnection` interfaces no longer
reflect the real purpose. New `Commands` interfaces provide the same
functionality and are located in `api.sync` and `api.async` packages
(respective `cluster.api.sync`, `cluster.api.async` and so on for the Redis Cluster client and PubSub).

The following interfaces are deprecated and substituted by new `...Commands` interfaces:

* RedisHashesAsyncConnection
* RedisHashesConnection
* RedisHLLAsyncConnection
* RedisHLLConnection
* RedisKeysAsyncConnection
* RedisKeysConnection
* RedisListsAsyncConnection
* RedisListsConnection
* RedisScriptingAsyncConnection
* RedisScriptingConnection
* RedisSentinelAsyncConnection
* RedisServerAsyncConnection
* RedisServerConnection
* RedisSetsAsyncConnection
* RedisSetsConnection
* RedisSortedSetsAsyncConnection
* RedisSortedSetsConnection
* RedisStringsAsyncConnection
* RedisStringsConnection
* RedisClusterConnection
* RedisConnection
* RedisClusterAsyncConnection
* RedisAsyncConnection
* BaseRedisConnection
* BaseRedisAsyncConnection

See https://github.com/mp911de/lettuce/wiki/Migration-from-3.x-to-4.x#optional-changes
for the migration matrix.

## New API's

* StatefulClusterConnection
* StatefulRedisPubSubConnection
* StatefulClusterConnection
* StatefulRedisSentinelConnection
* RedisPubSubAsyncConnection and RedisPubSubConnection

## API Changes

* readOnly and readWrite changed from `String` return type to `RedisFuture<String>`. The connection state is maintained by the future completion.
* Moved `CommandOutput` from `com.lambdaworks.redis.protocol` to  `com.lambdaworks.redis.output`
* Moved `SetArgs` from `com.lambdaworks.redis.protocol` to `com.lambdaworks.redis`
* All connections are `AutoCloseable` so you can handle connections using try-with-resources.
* `RedisFuture`s are based on `CompleteableFuture` and throw now any occurred exception when accessing the value using `get()`.
Exceptions are passed down the `CompletionStage`s.


Cross-slot command execution
----------------------------
Regular Redis Cluster commands are limited to single-slot keys, basically either single key
commands or multi-key commands that share the same hash slot of their keys.

The cross slot limitation can be mitigated by using the advanced cluster API for some
multi-key commands. Commands that operate on keys with different slots are decomposed into multiple commands.
The single commands are fired in a fork/join fashion. The commands are issued concurrently to avoid
synchronous chaining. Results are synchronized before the command is completed (from a user perspective).

Following commands are supported for cross-slot command execution:

* DEL: Delete the KEYs from the affected cluster. Returns the number of keys that were removed
* MGET: Get the values of all given KEYs. Returns the values in the order of the keys.
* MSET: Set multiple key/value pairs for all given KEYs. Returns always OK.

Cross-slot command execution is available on the following APIs:

* RedisAdvancedClusterCommands
* RedisAdvancedClusterAsyncCommands
* RedisAdvancedClusterReactiveCommands

Node Selection API/Execution of commands on multiple cluster nodes
------------------------------------------------------------------
The advanced cluster API allows to select nodes and run commands on the node selection. This API is subject
to incompatible changes in a future release. The API is exempt from any compatibility guarantees made
by lettuce. The current state implies nothing about the quality or performance of the API in question,
only the fact that it is not "API-frozen.". All commands are sent concurrently to the Redis nodes, meaning you do
not have to wait until the commands have finished to trigger the next command. That behavior is independent of the
API you're using. The Node Selection API is available from the synchronous and asynchronous
command interfaces:

**Asynchronous**

```java
RedisAdvancedClusterAsyncCommands<String, String> async = clusterClient.connect().async();
AsyncNodeSelection<String, String> slaves = connection.slaves();

AsyncExecutions<List<String>> executions = slaves.commands().keys("*");
executions.stream().forEach(result -> result.thenAccept(keys -> System.out.println(keys)));
```

**Synchronous**

```java
RedisAdvancedClusterCommands<String, String> async = clusterClient.connect().sync();
NodeSelection<String, String> slaves = connection.slaves();

Executions<List<String>> executions = slaves.commands().keys("*");
executions.stream().forEach(value -> System.out.println(value));
```

Commands are dispatched to the nodes within the selection, the result (CompletionStage or the value) is available
from the `AsyncExecutions`/`Executions` result holders.
This API is a technical preview, so your feedback is highly appreciated.

Read more: https://github.com/mp911de/lettuce/wiki/Redis-Cluster-(4.0)


Updated dependencies
--------------------
* rxjava 1.0.13 (new)
* Google Guava 17.0 -> 18.0
* netty 4.0.28.Final 4.0.30.Final
* commons-pool2 2.2 -> 2.4.2


Enhancements
------------
* Advanced Cluster API (async) #78
* Implement synchronous multi-node execution API enhancement #129
* Improve HLL command interface #77
* Move segregated API interfaces to own packages  #76
* Provide a stateful Redis connection and decouple sync/async API from connection resources #75
* Reactive support #68
* Pipelining for certain cluster commands #66
* Drop support for Java 6 and Java 7 #50
* Migrate RedisFuture to CompletionStage #48
* Allow limiting the request queue size #115
* Improve Codec API #118
* Allow to read from master/slave/nearest node when using Redis Cluster #114


Fixes
-----

Other
------
* Update Dependencies for lettuce 4.0 #116


lettuce requires a minimum of Java 8 to build and run. It is tested continuously against the latest Redis source-build.

For complete information on lettuce see the websites:

* http://github.com/mp911de/lettuce
* http://redis.paluch.biz