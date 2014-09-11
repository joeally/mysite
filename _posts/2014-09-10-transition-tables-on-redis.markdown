---
layout: post
title:  "Markov Sequences: Storing the Transition Tables on Redis (Part 4)"
date:   2014-09-10 12:00:00
categories: clojure programming 
---
 
In this post I will demonstrate how one could store their transition table on the [Redis](http://redis.io/) key value store.

##Why Redis?
Redis is simple, blazing fast and has a range of different data types available to the user. It does sacrifice the consistency guarantees the fully-fledged relational databases have but that's not something that is particularly desirable for this case.

##How will the Transition Table will be Structured in Redis?
Redis is key-value data store. The beauty of Redis is that values have data types (rather than just strings like in memcached). Each data type has its own strengths and weaknesses. I'll talk more about the particular data type shortly. First, I will define what they keys will be. 

###The Keys
Each key will represent the from vector just like it did when we were implementing the transition table protocol over Clojure's persistent map type. So if $$n=2$$ then the key will be a vector of length 2. Unlike Clojure's map data structure, Redis only supports keys that are strings. Fortunately this isn't to much of an issue as we can serialize the vector. For vectors of strings and other simple data types this is a relatively efficient operation.

###The Values
The values for our the persistent map implementation of the transition table protocol were my roughly defined multisets. Essentially the values were a map of symbols being transitioned to, to how many times they had been transitioned to. We could implement this on Redis as it does have a map type. However, in order for us to carry out a probabilistic random selection from that map we'd have to retrieve the entire contents of that map before making the random selection. We can certainly do better.

Redis doesn't have a multiset type, but it does have a set type (where the items of the set are strings). As I'm sure your aware a set cannot contain duplicate values. However the items of the sets are strings so one can simply add something to each item of the set to ensure its uniqueness. Then when we are returning values back from the string we can take of the extraneous information. Redis has the <code>SRANDMEMBER</code> command which selects a random member from a set. Once the uniqueness information is stripped of the randomness will be in line with the probability distribution for which the transitions occurred.

One last question remains. What bit of information can we add on to ensure that the member of the set is unique? A count of the number of transitions added to the transition table can be maintained and the current count can be appended onto a member of the set at the time that it is added.

###Executing that as Redis Commands
Redis supports Lua as a scripting language which allows us to perform relatively complex operations on the server itself. This is more efficient than sending commands to the server and then sending more commands based the result of the first command. The Lua command can be seen below

{% highlight lua %}
return redis.call('sadd', KEYS[1], string.format('%s:%d', KEYS[2], redis.call('incr', 'markovcount')))
{% endhighlight %}

<code>KEYS[1]</code> and <code>KEYS[2]</code> are arguments which can be passed into the command. In this case we use them to represent the key, which represents the from vector in the transition, and the set member which represents the destination of the transition respectively.

The Redis <code>SADD</code> commands adds a member to a set whilst <code>INCR</code> increases the integer value of a key and returns it. One can see that the code adds a member to the relevant set and appends a count to the end of it after a colon. 

Then to retrieve a value from the server one can simply call <code>SRANDMEMBER {key}</code>, split the result by the colon and take the first value of the resulting sequence. I probably could do this in a single Lua script but I'd rather stay in Clojure as much as possible as I am more familiar with it.

##Doing this from Clojure
Clojure is able to interface with Redis using the wonderful [carmine](https://github.com/ptaoussanis/carmine) library. It provides access to all of Redis' commands with a simple interface. In addition to this it automatically serialises and de-serialises Clojure data structures into strings. Which as I mentioned earlier is something that needs to happen as the keys to the transition table are Clojure vectors.

Below is the extension of the <code>TransitionTable</code> protocol over a record that defines a connection to a Redis key-value store.

{% highlight clojure %}
(ns markovtext.redis
  (:gen-class)
  (:require
   [markovtext.TransitionTable :as tt]
   [taoensso.carmine :as car :refer (wcar)]))

(defrecord RedisConn [pool spec key-options])

(defmacro wcar* [server-conn & body] `(car/wcar ~server-conn ~@body))

(def lua-add-string 
  (str "return redis.call('sadd', KEYS[1], string.format('%s:%d', KEYS[2], redis.call('incr', 'markovcount')))"))

(defn- gen-key [key key-options]
  (merge key-options {:key key}))

(defn- parse-key [key]
  (:key key))

(extend-protocol tt/TransitionTable
  RedisConn
  (rand-next [tt prev]
    (-> (wcar* tt (car/srandmember (gen-key prev (:key-options tt))))
        (clojure.string/split #":")
        (first)))
  (rand-first [tt]
    (parse-key (wcar* tt (car/randomkey))))
  (add-transition [tt from to]
    (wcar* tt (car/eval lua-add-string 2 (gen-key from (:key-options tt)) to))
    tt))
{% endhighlight %}

Notice that one can use the <code>gen-key</code> and <code>parse-key</code> functions to add extra information to the keys (helpful if the same Redis database is being used by more than one application). Note that <code>wcar*</code> <span style="display:none">*</span> is a macro that allows us to easily chain together Redis operations.

As described in the previous sections <code>rand-next</code> gets a random member of the set corresponding to the specified key with the <code>car/srandmember</code> call, splits by ":" and takes the first value. The <code>rand-first</code> function simply uses Redis' <code>RANDOMKEY</code> command using carmine's <code>car/randomkey</code> call. And <code>add-transition</code> is implemented by executing the Lua script defined previously.

##Actually Using the Redis Transition Table
Because the all of our functions only rely on the fact that the TransitionTable protocol is implemented by the data structure passed in, all of our existing functions will work out of box with a Redis database so long as a correctly configured <code>RedisConn</code> record is passed in as opposed to having a persistent map passed in. And for a locally hosted Redis database this can simply be specified with <code>(RedisConn. nil nil {})</code>


##What Next?
I was going to include a little application using Clojure's [<code>core.async</code>](https://github.com/clojure/core.async/) framework in this blog post. But I feel this post is dragging on so I'll develop it some more and write a blog post dedicated to it.
