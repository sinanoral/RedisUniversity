
# Stackexchange.Redis




### Connection Multiplexing

The most fundamental architectural feature of StackExchange.Redis is Connection Multiplexing. The library leans heavily on a class called the ConnectionMultiplexer. This class is responsible for arbitrating all connections to Redis, and routing all commands you want to send through the library through a single connection.

That's right, a single connection. The ConnectionMultiplexer opens exactly 2 connections per Redis Server, one of which is the interactive command connection to Redis, the other being the subscription connection for the pub/sub API which we'll explore later.

Like everything else in computing and software development, this approach has both benefits and tradeoffs.

#### Benefits

The ConnectionMultiplexer has proven to be extremely performant and robust. It's a powerhouse for pushing commands through to Redis.

* The single connection multiplexer matches the cardinality of Redis Threads. There's only one command thread in Redis, so sending additional commands concurrently doesn't help much as they will be waiting to be serviced by the command thread.
* It minimizes the number of sockets your application needs to open and maintain and wards off Socket Exhaustion.
* The multiplexer will maximize usage of your sockets, and automatically pipeline commands sent concurrently.

#### Tradeoffs

As with every other endeavour in software development, any architectural choice as profound as the ConnectionMultiplexer has some tradeoffs. We'll explore these in more detail throughout the course, but here are they are at a top level.

* Head-of-line blockages can occur with large payloads blocking out other requests.
* Blocking commands, e.g. blocking stream reads, the blocking list/sorted set commands cannot be used. This is because blocking the interactive connection will block out threads trying to use the connection concurrently.
* Transactions work a bit differently, we'll talk about them later in the course, but they are a tad different in that they don't fully support watches, and no command in a transaction is dispatched to Redis until execution time.
## Connect to Redis

There are two methods that you can use to connect to Redis: ConnectionMultiplexer.Connect and ConnectionMultiplexer.ConnectAsync - either will work. You'll need the parameters that you collected at setup:

* hostname
* port
* password (if on Redis Cloud or otherwise password protected)
* username (if not default)

There are essentially two overloads, one taking a connection string, and the other taking an instance of the ConfigurationOptions class, which is definitely easier to organize, but the connection string option is often easier for a simple connection.

#### Connection String

The Connection string is just a comma delimited string of different configuration paramaters, any hostname:port formatted argument in the connection string is treated as it's own endpoint, while any param=val formatted argument is treated as a parameter value. So the connection string:

#### redis-1:6379,password=foobar

Equates to StackExchange.Redis connecting to the host "redis-1", port 6379, and using the password "foobar" as it's password.

#### ConfigurationOptions

The ConfigurationOptions class allows for parameterized construction of a collection of options for the multiplexer. The equivalent options to our connection string above would be:

    var options = new ConfigurationOptions
    {
        EndPoints = new EndPointCollection{"redis-1:6379"},
        Password = "foobar"
    };

Ping Redis

After you've established a connection to Redis. Let's send our very first interactive command to Redis!

We'll start off by sending the simplest of Redis commands, PING. First, you'll need to get a handle to an IDatabase, the IDatabase is the main interactive command interface for Redis Commands. You can grab it from the Multiplexer by calling the GetDatabase method on the ConnectionMultiplexer.

With that done, all you need to do to ping Redis is call Ping, print the results, and you're done!


## Connect to Redis Solution

The following is what my code looked like after the previous exercise, yours might be slightly different, and that's ok! but fundamentally, you should have:

* Connected to the ConnectionMultiplexer
* Grabbed an IDatabase from the Multiplexer
* Called Ping on the IDatabase
* Printed the results from that ping.
   
```
    // Start Programming Challenge
    using StackExchange.Redis;
    Console.WriteLine("Hello Redis!");
    
    var muxer = ConnectionMultiplexer.Connect(new ConfigurationOptions
    {
        EndPoints = new EndPointCollection{"localhost:6379"}
    });
    
    var db = muxer.GetDatabase();
    var res = db.Ping();
    Console.WriteLine($"The ping took: {res.TotalMilliseconds} ms");
    //End Programming Challenge 
```



        

## The Interfaces of StackExchange.Redis

The public API of StackExchange.Redis is broken up across several critical interfaces. We'll briefly go over each of them in this section. You've actually already touched two of them IConnectionMultiplexer and IDatabase.

The Interfaces

* IConnectionMultiplexer
* IDatabase
* IServer
* ISubscriber
* ITransaction


## IConnectionMultiplexer

The IConnectionMultiplexer is responsible for maintaining all of the connections to Redis. As I described in the previous section. It routes all the commands to Redis through a single connection for interactive commands, and a separate connection for subscription, which we'll discuss more in depth later.

The IConnectionMultiplexer is responsible for exposing a simple interface to get other critical interfaces of the library. Including the IDatabase, ISubscriber, and IServer.
## IDatabase

The IDatabase can be thought of as the primary interactive interface to Redis. It provides a single interface for your entire Redis Instance, and is the preferred interface when you are executing single commands that manipulate your application's data to Redis.

The IDatabase, unlike the IServer, abstracts the particulars of your Redis deployments away. Consequentially, if you are running in a cluster and are preforming a write, the IDatabase does not require you to know which server in particular you need to write to. Also, if you have many replicas per master shard in your Cluster or Sentinel Redis deployments, the IDatabase will leverage the ConnectionMultiplexer to automatically distribute your reads across your deployment.

## IServer

The IServer is an abstraction to a single instance of a Redis Server. You can grab an instance of an IServer by using the IConnectionMultiplexer.GetServer command, passing in the exact endpoint information you want to retrieve.

IServer has a fundamentally different role than IDatabase as you're going to use it to handle the server level commands. That means that in general, data modeling commands are not appropriate to be used on a server. Rather operations like checking the server's info (the basic info of Redis), it's configuration, updating it's configuration, checking memory statistics, and the like are IServer operations. Even scanning for the keys of a Redis server should be done at the server level.

## ISubscriber

The ISubscriber is the interface responsible for maintaining subscriptions to Redis in the pub/sub interface. Unlike the other interfaces we've looked at thus far, the subscriber does not leverage the interactive connection.

The Multiplexer explicitly opens a separate connection for subscriptions because when you subscribe to any channel on a client in Redis, the client connection converts to subscription mode. This limits the connection to only use commands that implement subscriber functionality.

True to it's name however, the Multiplexer continues to maintain a single connection per server, and all subscriptions are handled on that single connection.

You you can get an instance of an ISubscriber by calling the IConnectionMultiplexer.GetSubscriber() method.

## ITransaction

The ITransaction provides an interface for Redis Transactions. Transactions in Redis differ from transactions in other databases, for a full description of transactions check out the transaction section of RU101.

The ITransaction interface is fundamentally an async interface. It exposes a very similar command set to the IDatabase, but it will only expose async versions of each command. That is because each command in ITransaction is async, as they will not be competed until after Execute is called. Only after the Execute is called can the underpinning tasks for the Transaction be awaited.

You can get an instance of an ITransaction by calling IDatabase.GetTransaction() on your IDatabase object.