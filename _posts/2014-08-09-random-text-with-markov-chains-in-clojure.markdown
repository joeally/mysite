---
layout: post
title:  "Random Text with Markov Chains in Clojure"
date:   2014-06-28 12:00:00
categories: clojure markov
---

In my previous post gave a brief introduction behind the theory of generating random text using markov chains. In this post I outline how I implemented this in clojure.

## The Input
The inputs in this particular example will be a collection of sequences. In this particular blog post I each item in a particular sequence will be a string representing a word. But there is no reason why any type of symbol could not be used instead. 

However it is difficult to collections of sequences of words. It is more common to find a corpus of documents similar to the corpus shown below (in clojure a data structure).

{% highlight clojure %}
'("To succeed in life, you need two things: ignorance and confidence"
  "The secret of getting ahead is getting started"
  "All generalizations are false, including this one.")
{% endhighlight %}

The below function converts a corpus into a collection of sequences of words. It converts all the characters to lower case and splits the corpus in to sequences of tokens where each token is seperated by a space and common punctution characters such as commas, full-stops, colons and semicolons are their own tokens

{% highlight clojure %}
(defn process-corpus [corpus]
  (for [doc corpus]
    (-> doc
         (clojure.string/lower-case)
         (clojure.string/replace #"[\.,?:;]" #(clojure.string/join [" " (str %)]))
         (clojure.string/split #" "))))
{% endhighlight %}

This function converts the corpus shown previously into the following clojure datastructure:
{% highlight clojure %}
(["to" "succeed" "in" "life" "," "you" "need" "two" "things" ":" "ignorance" "and" "confidence"]
 ["the" "secret" "of" "getting" "ahead" "is" "getting" "started"]
 ["all" "generalizations" "are" "false" "," "including" "this" "one" "."])
{% endhighlight %}

## Building the Transition Table
From the previous post you'll recall that to build any kind of Markov chain a tranistion table is required. Clojure has a very powerful builtin persistant map type which excels in situations such as these. The keys will represent what we are transitioning from and the values will be what we are transitioning to. To enable markov chains with $$n > 1$$, the keys of the map will be vectors of length $$n$$.
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
          ;; e.g [("to" "succeed" "in") ("succeed "in" "life") ("in "life" ",") ...]
          (partition (inc n) 1)
          ;;group sequences into a map by the first n items in each sequence
          ;; e.g {["to" "succeed] (("to" "succeed" "in")), ["succeed" "in"] (("succeed" "in" "life")) ..}
          (group-by #(vec (take  n %)))
          ;;have the values to be only the last element of the n+1 sequence
          ;; e.g {["to" "succeed] ("in"), ["succeed" "in"] ("life") .. }
          (map (fn [[k v]] [k (map last v)]))
          ;;collect transition vectors into a map
          (reduce (fn [tt [from to-vec]] (tt/add-transition-vec tt from to-vec)) {})))
  ([seqn] (collect-transitions seqn 1)))
{% endhighlight %}
