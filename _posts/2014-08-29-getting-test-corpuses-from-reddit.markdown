---
layout: post
title:  "Markov Sequences: Getting Test Corpuses from Reddit (Part 3)"
date:   2014-08-29 12:00:00
categories: clojure markov reddit
---

In my [previous]({{page.previous.url}}) post I provided a solution for creating plausible random strings of text using Clojure. But our random sequence generator needs to be trained on some data and we don't yet have any. So I have written a short blog post about getting some test data from the [reddit](http://www.reddit.com) social media website.

##Why Choose Reddit?
Reddit has a very accessible [API](http://www.reddit.com/dev/api) which gives us access to pretty much everything on reddit. Reddit also has a vast amount of content posted on it each day, making it perfect for collecting data.

##The Data
The data I will be collecting from reddit will be comments. These can easily be accessed using the urls of the following form:

{% highlight html %}
http://www.reddit.com/r/{subreddit}/comments.json
{% endhighlight %}

Where <code>{subreddit}</code> is replaced by the name of the subreddit we are retrieving comments for.

##Retrieving the Data
Retrieving the data is performed using the excellent [clj-http](https://github.com/dakrone/clj-http) library. Because of this all functions defined below are in the following namespace (unless specified otherwise).

{% highlight clojure %}
(ns markovtext.reddit
  (:gen-class)
  (:require
   [clj-http.client :as client]
   [clojure.string :as string])
  (:import [java.net URLEncoder]))
{% endhighlight %}

Using [clj-http](https://github.com/dakrone/clj-http) I have written the following function which creats HTTP GET requests and returns a Clojure data structure parsed from the JSON in the response of the request.

{% highlight clojure %}
(defn urlopen [url data cookie]
  (let [response (client/get (string/join [url "?" (format-data data)])
                             {:headers {"User-Agent" "reddit.clj"}
                              :cookies cookie
                              :as :json
                              :socket-timeout 10000
                              :conn-timeout 10000})]
    (if (= 200 (:status response))
      (:body response)
      nil)))
{% endhighlight %}

With the <code>format-data</code> function being defined as below:

{% highlight clojure %}
(defn format-data [data]
  (->> data
       ;; Convert keys and values to suitable form
       (map (fn [[k v]] [(name k) (URLEncoder/encode (str v))]))
       ;; Join up pairs with "="
       (map #(string/join #"=" %))
       ;; Join the kv pairs
       (string/join #"&")))
{% endhighlight %}

We are now ready to make requests to reddit to get data. The following function makes a request to a url (will probably only return something useful for a reddit url):

{% highlight clojure %}
(defn get-listing [url after]
  (->> (urlopen url (if (nil? after) {:limit 1000} {:limit 1000 :after after}) nil)
       ;; the interesting bits are in the data field
       (:data)))
{% endhighlight %}

Because reddit limits the number of items returned in a single RESTful request, to gather large numbers of comments, we need to make multiple request. However reddit also rate limits requests at 30 requests per minute. I've written a function (shown below) which allows one to make an unbounded number of requests to reddit without breaking the API rules.

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

This function makes use of the <code>lazy-cat</code> function in <code>clojure.core</code> to create a lazy sequence of reddit listings. Note that I use the rather hacky <code>Thread/sleep</code> to ensure complicity with the API rules and avoid rate limiting. Using a constant sleep time means that requests are being made at a slightly slower rate than allowed by reddit. Although not perfect, this solution will suffice for now.

Now we can get listings, we can now start retrieving comments. Below the <code>reddit-corpus</code> function is defined along with a couple of helper functions. <code>reddit-corpus</code> returns a lazy sequence of comment bodies. That is, it strips all other information from each reddit listing other than the body of the comment.

{% highlight clojure %}
(defn process-reddit [st]
  (-> st
      (string/replace #"\((http://)?[\w\.\?&]+\)" "")
      (string/replace #"[\*\(\)\[\]]" "")
      (string/replace #"\d\)" "")))

(defn sr-comments [sr]
  (map
   :data
   (get-listings (string/join [redditurl "/r/" sr "/comments.json"]))))

(defn reddit-corpus [sr]
  (map
   (comp process-reddit :body)
   (sr-comments sr)))
{% endhighlight %}


## Generating Random Sequences from this Data
For this section assume that all of the above functions are imported under the <code>reddit</code> namespace. Also one should assume that the <code>TransitonTable</code> protocol and its extension over Clojure's map (which were defined in the previous blog post) are in imported under the <code>tt</code> name space.

Using the functions defined in the [previous blog post]({{page.previous.url}}) one can open up the REPL and run the following code to generate a random sequence of words based on a transition table created from reddit comments.

{% highlight clojure %}
> (def corpus (reddit/reddit-corpus "technology"))
> (take 20 (corpus-likely-seq (take 1000 corpus) 1))
{% endhighlight %}

This will generate a sequence of twenty words using a Markov chain where $$n=1$$ using 1000 reddit comments from the technology subreddit. Using the above code the following sequence was generated

{% highlight clojure %}
("alterior" "motive" "," "and" "potential" 
"solution" "."  "npd" "," "they're" "trying" 
"to" "be" "considered" "non-trivial" "since" "your" "mother" "," "so")
{% endhighlight %}

We can then turn this sequence into a single string using the following function which ensures that punctuation is properly spaced:
{% highlight clojure %}
(defn stringify-seq [seq]
  (->>
   seq
   (replace { "i" "I"})
   (clojure.string/join " ")
   (#(clojure.string/replace % #" +[\.\?,]" (fn [s] (str (second s)))))))
{% endhighlight %}

When running this function on the returned sequence one will get something which looks like what is shown below.

> "alterior motive, and potential solution. npd, they're trying to be considered non-trivial since your mother, so"

As you can see the quote doesn't make much sense, although it does role off the tongue reasonably well. To generate more realistic sentences we'll probably have to use higher values for $$n$$. For that we'll need a much larger transition table (the number of keys in the table to the power $$n$$).

In the next blog post I will look at continually updating the transition table and storing them in a [redis](http://redis.io/) key-value store to allow us to better scale things up.
