---
layout: post
title:  "Markov Sequences: Generating Plausible Text (Part 5)"
date:   2014-09-21 12:00:00
categories: clojure programming 
---

In this post I've finally gotten to the stage where I can now write an application for generating plausible sentences. Things are a little more difficult than they first seem. Mainly because I'd like to generate a plausible random text from multiple subreddits. The rate limiting that reddit employs on its API (30 requests per minute) makes it rather tricky to pull data from multiple reddits simultaneously. 

##Fixing get-listings
Remember that the <code>get-listings</code> function looked rather like this.

{% highlight clojure %}
(defn get-listings
  ([url after]
     (Thread/sleep 2000)
     (let [listing (get-listing url after)]
       (lazy-cat (:children listing) (get-listings url (:after listing)))))
  ([url]
     (let [listing (get-listing url nil)]
       (lazy-cat (:children listing) (get-listings url (:after listing))))))
{% endhighlight %}

This is not thread safe at all. If multiple threads are calling this function there will be more than 30 requests per minute.

There needs to be some way of co-ordinating different threads making requests.

###Enter core.async
[<code>core.async</code>](https://github.com/clojure/core.async) is library which provides asynchronous programming. It is centred around the concepts of channels. Processes can post values into channels or take values from channels.

Using <code>core.async</code> we can spawn multiple processes using go block, each of which will post to the channel with the url it wants to request and a channel which can be used to post back to that process. A worker process will then read from that channel every 2 seconds, make a request based on the message read, and post the result to the channel sent in the message. That way a single process is responsible for making the request and can ensure that one is made only every 2 seconds.

The worker process is spawned by the function specified below (<code>core.async</code> has been required as <code>async</code>):
{% highlight clojure %}
(def last-requested (atom 0))
(def listings-chan (async/chan))
(def listings-worker-started (atom false))
(defn listings-worker []
  (async/go
   (while true
     (let [[url after chan] (async/<! listings-chan)
           pause (max
                  0
                  (- 2000 (- (System/currentTimeMillis) @last-requested)))]
       (Thread/sleep pause)
       (async/>! chan (get-listing url after))
       (reset! last-requested (System/currentTimeMillis))))))
{% endhighlight %}

The <code>async/go</code> macro is reffered to as a "go" block and internally spreads processes defined inside the go block across a pool of threads and "parks" the processes when they are waiting for channel rather than blocking. <code>&#60;!</code> takes from a channel and <code>&#62;!</code> puts into a channel.

We now track how long it has been since our last request using the <code>last-requested</code> atom to ensure that the thread is never paused for more than 2 seconds.

<code>get-listings</code> now simply puts requests into a channel and waits on a response from the worker threadbefore placing the data in the lazy sequence.

{% highlight clojure %}
(defn get-listings
  ([url after]
     (when (not @listings-worker-started)
       (listings-worker)
       (swap! listings-worker-started not))
     (let [chan (async/chan)]
       (async/go (async/>! listings-chan [url after chan]))
       (let [listing (async/<!! chan)]
         (lazy-cat (:children listing) (get-listings url (:after listing))))))
  ([url]
     (get-listings url nil)))
{% endhighlight %}

##Putting this all together
To put it all together we are going to use a producer-consumer architecture to retrieve content from the reddit API and put it into a Redis key value start. Below the producer and consumer functions are defined below.

{% highlight clojure %}
(defn producer [channel subreddit running]
  (let [corpus (process-corpus (reddit/reddit-corpus subreddit))]
    (async/go
     (loop [comment-seqs corpus]
       (if @running
         (let []
           (doseq [transition (col-transitions (first comment-seqs) 2)]
             (async/>! channel transition))
           (recur (rest comment-seqs)))
         nil)))))

(defn consumer [channel running redis-conn]
  (async/go
   (while @running
     (let [[from-vec to-symbol] (async/<! channel)]
       (tt/add-transition redis-conn from-vec to-symbol)))))

(def running (atom true))
{% endhighlight %}

Note that the producers and consumers only run so long as the <code>running</code> atom is true.

Now I need to pull this into a proper application in the <code>-main</code> function. The main will start some producers and a consumer and generate plausible text. The <code>-main</code> function is defined below.
{% highlight clojure %}
(defn -main
  "I don't do a whole lot ... yet."
  [& args]
  (let [reddit-chan (async/chan)
        redis-conn (RedisConn. nil nil {:app "markovtext" :group "functional langs" :n 2})]
    (producer reddit-chan "circlejerk" running)
    (producer reddit-chan "funny" running)
    (producer reddit-chan "wtf" running)
    (consumer reddit-chan running redis-conn)
    (while @running
      (let [msg (read-line)]
        (if (= msg "close")
          (do
            (async/close! reddit-chan)
            (swap! running not)
            (println "closed"))
          (println
           (stringify-seq
            (take 25 (quite-likely-seq (tt/rand-first redis-conn) redis-conn)))))))))
{% endhighlight %}

The main function first starts some producers and a consumer and then starts a loop which reads user input an prints out a generated sequence of words (length 25) unless the user inputs "close" when the <code>running</code> atom is set to false and the application exits. The application uses Markov chains with $$n=2$$ and gets its training set from the circlejerk, wtf and funny subreddits. Below is a sample bit of text which was generated after the application was left running for a little while.

>"you call your wife. you’re thucking dead, it's usually because they didn't know it sucks. but nobody actually sees 18 year old"

As one can see it's a little perverse an broadly nonsensical. But what else does one expect given the subreddits used to generate the transition tables.
