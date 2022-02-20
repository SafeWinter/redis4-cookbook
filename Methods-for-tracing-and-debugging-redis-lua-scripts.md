# ~~5~~ ~~6~~ 7 Methods For Tracing and Debugging Redis Lua Scripts

[Itamar Haber](https://redis.com/blog/author/itamar/)

December 2, 2014

![Debugging Redis Lua](assets/logo.png)

### Or: To Err is Human, To Fix Debug.

#### **Mother of all Updates:** Redis v3.2 features its very own [Lua debugger](http://redis.io/topics/ldb).

#### ***Update 1:** I’ve followed up on the topic with a vastly superior debugging method so check out part 2 – [redis-lua-debugger](https://github.com/Redis/redis-lua-debugger) and its [accompanying post](https://redis.com/blog/pop-the-red-boxs-lid-redis-lua-debugger/) <- **mini-update:** rld is no longer maintained as it isn’t compatible with v3+.*

If you’ve ever written even a single line of code, you know that Benjamin Franklin’s famous quote should be amended to:

> “In this world nothing is certain, except death, taxes … **and BUGS**.”

Software defects are a fact of life because software is made by fleshware, and humans err. Even if you are a good programmer who writes good code (or a great programmer who steals it), use proven methodologies and design patterns, employ only best-of-breed tools, and submit to peers’ code reviews… despite your best efforts, you’re likely to find yourself time and time again banging your head against the wall because of an elusive gremlin.

Tracking down these issues isn’t an easy task. It requires patience, effort and in many cases a touch of inspiration to correctly identify the root cause of a failure. When developing Lua scripts for Redis (a feature that’s available from version 2.6 onwards) this can become even trickier. That’s because your code runs within the server itself, making it much harder to gain visibility into the code’s innards. To make this somewhat easier (and perhaps save your wall from a few bangs), here are five ways you can gain Superman-like X-Ray vision into your Lua script.



```
Reminder
--------
You can run a Lua script in a file with `redis-cli` in the following manner:


  redis-cli --eval script.lua key1 key2 ... , arg1 arg2 ...
                       ^          ^         ^     ^
    the script's file -+          |         |     |
       keys passed to the script -+         |     |
          a comma separates keys from args -+     |
                  arguments passed to the script -+
```



### 1. Use the Redis Log

Redis’ embedded Lua engine provides a function that prints to its log file (configured with the loglevel and logfile directives in your redis.conf). This function, conveniently named redis.log, is dead simple to use – just call it from your script like so:



```lua
redis.log(redis.LOG_WARNING, "foo bar")
```



The redis.log function accepts two arguments: the first one is the message’s log level (choose being between LOG_DEBUG, LOG_VERBOSE, LOG_NOTICE and LOG_WARNING), and the second argument is the to-be-logged value. For more information, see [EVAL](http://redis.io/commands/eval)‘s documentation.

**Pros:** Using the log file is often the easiest way to trace your workflow. In addition, you can view the logged message in near real-time (i.e. by tailing the log file).

**Cons:** There are cases in which this approach isn’t a viable option (e.g. when you don’t have access to the Redis host or when the log is too noisy to work with comfortably). Don’t despair, however, because there’s more than one way to skin a cat (meeeeeow!?!).

### 2. Use a Lua Table

Lua’s tables are associative arrays and are the only “container” data structures of the language. You can easily use them to store messages in a log-like manner and return the resulting array once your code finishes running. Here’s an example:



```lua
local logtable = {}


local function logit(msg)
  logtable[#logtable+1] = msg
end


logit("foo")
logit("bar")


return logtable
```



Running the example above yields the following:



```
foo@bar:~$ redis-cli --eval log-with-table.lua
1) "foo"
2) "bar"
```



**Pros:** Works everywhere, and requires very little setup.

**Cons:** Returning the log table as your code’s reply prevents it from returning anything meaningful. This approach consumes memory to store the intermediate log table, and you need to wait for the script to finish before you can get the log messages.

### 3. Use a Redis List

If you require your Lua code to return a meaningful reply once it finishes execution and still retain a logging mechanism, you can use Redis’ List data structure for storing and retrieving your messages. Here’s a snippet that shows this approach:



```lua
local loglist = KEYS[1]
redis.pcall("DEL", loglist)


local function logit(msg)
  redis.pcall("RPUSH", loglist, msg)
end


logit("foo")
logit("bar")


return 42
```



Note that the script’s 2nd line deletes the log’s key for housekeeping. Running this script followed by an [LRANGE](http://redis.io/commands/lrange) to obtain the “log” results in the following:



```
foo@bar:~$ redis-cli --eval log-with-list.lua log
(integer) 42
foo@bar:~$ redis-cli LRANGE log 0 -1
1) "foo"
2) "bar"
```



**Pros:** Like Lua tables, this is straightforward and just works.

**Cons:** Similar to the tables approach plus there’s an arbitrary size limit (2^32) on the List’s length. You also need to pass the List’s key name to ensure cluster-compatibility and most importantly, since you’ll be performing a write operation to Redis this can’t be used with non-deterministic commands.

### 4. Use Redis’ Pub/Sub

[Pub/Sub](http://redis.io/topics/pubsub) has many wonderful uses, but did you know it can also be used for debugging? By having your script publish log messages to a channel and subscribing to that channel, you can keep track of what’s going on. Here’s how to do it:



Before running this script, open another terminal window and subscribe to your log channel. While the script is running, you should get the following output in your subscriber terminal:



```
foo@bar:~$ redis-cli SUBSCRIBE log
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "log"
3) (integer) 1
1) "message"
2) "log"
3) "foo"
1) "message"
2) "log"
3) "bar"
```



**Pros:** Little overhead, real-time display of messages.

**Cons:** Pub/Sub’s delivery isn’t guaranteed, so there’s a chance you could miss a revealing log message (but that’s really a long shot).

### 5. Use a Lua Debugger

Adding tracing to your code can only get you so far when you’re trying to grok a piece of code, because sometimes you really need/want a full-fledged debugger at your disposal. [This clever trick](http://www.trikoder.net/blog/make-lua-debugging-easier-in-redis-87/) provided by Marijan Šuflaj from [@Trikoder](https://twitter.com/trikoder) gives you exactly that – a debugger for your Lua scripts with Redis.

The gist of Marijan’s idea is using a freely-available Lua debugger to debug your Lua code and adding Redis-specific commands with thin wrapping code. His blog post takes you through the steps to accomplish that feat, and even includes sample code for the Redis-specific commands wrapper. While this doesn’t exactly let you debug the code in vivo, it’s the closest you can get and for all practical purposes is as good.

**Pros:** Use a full-blown debugger for your Lua code.

**Cons:** Requires a degree of non-trivial setup.

### **Update 2:** Bonus Method – Use

`MONITOR`and`ECHO`I’ve stumbled on another useful way for tracing that is very similar to using the log, but has an additional perk. The idea is to open a dedicated connection to your server and run the`MONITOR`command (note that this will return a stream of **all commands** executed by Redis so you don’t want to do this on a busy server). Once you have the monitor set up, you can call`ECHO`from the script to trace stuff (e.g.`redis.call('ECHO', 'the value of foo is' .. foo))`. The added perk is that your monitor stream also shows on every other`redis.call`that your script executes inline with your traces – neat-o!

### **Update 3**: Another Bonus Method – Use Lua’s print

Lua’s print command works just great in Redis’ script and you can call it at will and trace to your heart’s content. The only problem with it, however, is that it outputs everything directly to stdout, so unless you’re watching the server’s output you’re apt to miss it.

### Conclusion

There are many ways your code can (and will) fail, and tracking down the cause can be a daunting task. When it comes to developing Lua scripts for Redis, I hope you’ll find these methods useful in squashing the pesky critters that keep your code from running as you expect it to. Questions? Feedback? Any other tricks, tips or subjects that you’d like to see covered? [Email](mailto:itamar@redis.com) or [tweet](https://twitter.com/itamarhaber) me – I’m highly available 🙂