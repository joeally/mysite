---
layout: post
title:  "Random Text with Markov Chains in Clojure"
date:   2014-08-09 12:00:00
categories: clojure markov
---

In my [previous]({{page.previous.url}}) post I gave a brief introduction behind the theory of generating random text using Markov chains. In this post I outline how I implemented this in Clojure.

## The Input
The inputs in this particular example will be a collection of sequences. In this blog post, each item in a particular sequence will be a string representing a word. But there is no reason why any type of symbol could not be used instead. 

Howevever, rather than using a single document to build our random sequence generator we will use a corpus of documents. That way we can continuously improve our random sequence generator by adding the transitions from more documents to our transition table. An example of such a corpus is shown below (in a Clojure data structure).

{% highlight clojure %}
'("To succeed in life, you need two things: ignorance and confidence"
  "The secret of getting ahead is getting started"
  "All generalizations are false, including this one.")
{% endhighlight %}

The below function converts a corpus into a collection of sequences of words. It converts all the characters to lower case and splits the corpus in to sequences of tokens where each token is separated by a space and common punctution characters such as commas, full-stops, colons and semicolons are their own tokens

{% highlight clojure %}
(defn process-corpus [corpus]
  (for [doc corpus]
    (-> doc
         (clojure.string/lower-case)
         (clojure.string/replace #"[\.,?:;]" #(clojure.string/join [" " (str %)]))
         (clojure.string/split #" "))))
{% endhighlight %}

This function converts the corpus shown previously into the following Clojure data structure:
{% highlight clojure %}
(["to" "succeed" "in" "life" "," "you" "need" "two" "things" ":" "ignorance" "and" "confidence"]
 ["the" "secret" "of" "getting" "ahead" "is" "getting" "started"]
 ["all" "generalizations" "are" "false" "," "including" "this" "one" "."])
{% endhighlight %}

## Building the Transition Table
From the previous post you'll recall that to build any kind of Markov chain a tranistion table is required. Clojure has a very powerful builtin persistant map type which excels in situations such as these. The keys will represent what we are transitioning from and the values will be what we are transitioning to. To enable Markov chains with $$n > 1$$, the keys of the map will be vectors of length $$n$$.
In the previous post I mentioned that the values of the map would be lists of words. As we do not need any notion of ordering we can store this list more efficiently as a map of words to the number of times they were transitioned to. Otherwise known as a *bag-of-words*.

The transition table for $$n = 2$$ for the following sequence
{% highlight clojure %}
(def sentence-seq ["to" "succeed" "in" "life" "," "you" "need" "two" "things" ":" "ignorance" "and" "confidence"])
{% endhighlight %}

can be seen below.

{% highlight clojure %}
{["," "you"] {"need" 1},
 ["in" "life"] {"," 1},
 ["two" "things"] {":" 1},
 [":" "ignorance"] {"and" 1},
 ["ignorance" "and"] {"confidence" 1},
 ["succeed" "in"] {"life" 1},
 ["need" "two"] {"things" 1},
 ["life" ","] {"you" 1},
 ["you" "need"] {"two" 1},
 ["things" ":"] {"ignorance" 1},
 ["to" "succeed"] {"in" 1}}
 {% endhighlight %}

One can achieve this result by collecting the transitions using <code>col-transitions</code> and converting them into a transition table using <code>make-transition-table</code>. The two functions are shown below with an example of how they would typically be composed.

{% highlight clojure %}
(defn col-transitions [seqn n]
  (partition (inc n) 1 seqn))

(defn make-transition-table [transitions]
  (->> transitions
      ;;convert transition to [from-vec to-symbol] pairs
      (map (fn [part] [(vec (take (dec (count part)) part)) (last part)]))
      ;;collect transition vectors into a map
      (reduce (fn [tt [from to]] (tt/add-transition tt from to)) {})))

(def transiton-table 
 (make-transition-table 
   (col-transitions sentence-seq 2)))
{% endhighlight %}


Above you can see the tt/add-transition function which I have not defined yet. This function simply adds a transition to the transition table. The <code>from</code> argument is the token being transitioned from and the <code>to</code> argument is the token being transitioned to. <code>tt</code> is the namespace given to the pacakge in which the helper functions for the transition table are defined. Below is the file in which the <code>add-transition</code> function is defined.

{% highlight clojure %}
(ns markovtext.transitiontable
  (:gen-class))

(defn concat-bag [bag1 bag2]
  (merge-with + bag1 bag2))

(defprotocol TransitionTable
  "A protocol for a transition table"
  (rand-next [tt prev])
  (rand-first [tt])
  (concat-tt [tt1 tt2])
  (add-transition-vec [tt from to-vec])
  (add-transition [tt from to]))

(extend-protocol TransitionTable
  clojure.lang.IPersistentMap

  (add-transition [tt from to]
    (update-in tt [from to] (fnil inc 0)))

  (add-transition-vec [tt from to-vec]
    (reduce #(add-transition %1 from %2) tt to-vec))

  (rand-first [tt]
    (rand-nth (keys tt)))

  (rand-next [tt prev]
    (try
      (rand-nth (apply concat (for [[k n] (get tt prev)] (repeat n k))))
      (catch IndexOutOfBoundsException e nil))))
{% endhighlight %}

One can see that the function is defined as part of a Clojure [protocol](http://clojure.org/protocols). This will allow me to easily specify another implementation should I wish to.

For the case when we want to build a transition table for a whole corpus I have written the function below. One can simply concatenate the sequences of transitions for the documents in the corpus into a single sequence before using that to construct the transition table.

{% highlight clojure %}
(defn corpus-transitions
  ([corpus n]
      (->> corpus
           (map #(col-transitions % n))
           (apply concat)
           (make-transition-table)))
  ([corpus] (corpus-transitions corpus 1)))
{% endhighlight %}

##Generating the Sequence
Now we've built the transition table we can now generate our sequences of tokens. In addition to the transition table, the below function takes a vector of first $$n$$ tokens for the sequence (the initialization sequence) from which we can generate the subsequent items in the sequence using the transition table.

{% highlight clojure %}
(defn quite-likely-seq [fst trans-tbl]
  (map
   first
   (iterate
    (fn [prev]
      ;;return a sequence of all except the first value in the previous result and the next in the sequence
      (conj (vec (drop 1 prev)) (tt/rand-next trans-tbl prev)))
    fst)))
{% endhighlight %}

We are using the [iterate](http://clojuredocs.org/clojure_core/clojure.core/iterate) function to generate a lazy sequence of from part of the transitions. That is it will generate sequence of pairs in the following form if $$n=2$$

{% highlight clojure %}
;;tn represents the nth token in the randomly generated sequence
[t0 t1] [t1 t2] [t2 t3] [t3 t4] ...
{% endhighlight %}

or the following if $$n=3$$:
{% highlight clojure %}
;;tn represents the nth token in the randomly generated sequence
[t0 t1 t2] [t1 t2 t3] [t2 t3 t4] [t3 t4 t5] ...
{% endhighlight %}

We can then take the take the first of each of these to generate our random sequence. So the above sequences will be result in the following sequence:

{% highlight clojure %}
;;tn represents the nth token in the randomly generated sequence
t0 t1 t2 t3 ...
{% endhighlight %}

We can then create the following convenience function that takes a corpus and generates a sequence (rather than taking a transition table and an initialization sequence).

{% highlight clojure %}
(defn corpus-likely-seq [corpus n]
  (let [transitions (corpus-transitions (process-corpus corpus) n)]
    (let [start (rand-nth (vec (keys transitions)))]
      (quite-likely-seq start transitions))))
{% endhighlight %}

In the next blog post I'll talk about scrapping some data from reddit and generating some random sequences using it.
