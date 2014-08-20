---
layout: post
title:  "Random Text with Markov Chains in Clojure"
date:   2014-08-09 12:00:00
categories: clojure markov
---

In my [previous]({{page.previous.url}}) post I gave a brief introduction behind the theory of generating random text using Markov chains. In this post I outline how I implemented this in Clojure.

## The Input
The inputs in this particular example will be a collection of sequences. In this particular blog post In each item in a particular sequence will be a string representing a word. But there is no reason why any type of symbol could not be used instead. 

However it is difficult to collect of very sequences of words. It is more common to find a corpus of documents similar to the corpus shown below (in Clojure a data structure).

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
["to" "succeed" "in" "life" "," "you" "need" "two" "things" ":" "ignorance" "and" "confidence"]
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

The function which produces this result is as follows:

{% highlight clojure %}
(defn collect-transitions
  ([seqn n]
     (->> seqn
          ;;split sequence into overlapping sequences of n+1
          (partition (inc n) 1)
          ;;convert these sequnces into vector in the form [fromvec to]
          (map (fn [part] [(vec (take n part)) (last part)]))
          ;;collect transition vectors into a map
          (reduce (fn [tt [from to]] (tt/add-transition tt from to)) {})))
  ([seqn] (collect-transitions seqn 1))
{% endhighlight %}

Above you can see the tt/add-transition function which I have not defined yet. This function simply adds a transition to the transition table. The "from" argument is the token being transitioned from and the "to" argument is the token being transitioned to. "tt" is the namespace I am importing the package containing this function under. Below is the file in which the "add-transition" function is defined:

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

  (concat-tt [tt1 tt2]
    (merge-with concat-bag tt1 tt2))

  (rand-first [tt]
    (rand-nth (keys tt)))

  (rand-next [tt prev]
    (try
      (rand-nth (apply concat (for [[k n] (get tt prev)] (repeat n k))))
      (catch IndexOutOfBoundsException e nil))))
{% endhighlight %}

One can see that the function is defined as part of a Clojure [protocol](http://clojure.org/protocols). This will allow me to easily specify another implementation should I wish to.

##Generating the Sequence

Now we are able to build a transition table we can now generate our sequences of tokens.  

{% highlight clojure %}
(defn quite-likely-seq [prev trans-tbl n]
  (map
   first
   (iterate
    (fn [prev]
      ;;return a sequence of all except the first value in the previous result and the next in the sequence
      (conj (vec (drop 1 prev)) (tt/rand-next trans-tbl prev)))
    prev)))
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

In the next blog post I'll talk about scrapping some data from reddit and generating some random sequences using it.
