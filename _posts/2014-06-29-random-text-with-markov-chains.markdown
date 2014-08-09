---
layout: post
title:  "Generating Random Text with Markov Chains"
date:   2014-06-29 12:00:00
categories: clojure programming 
---

One method of generating random text is to model text as a [discrete time Markov chain](http://en.wikipedia.org/wiki/Markov_chain). Markov chains model sequential events. A Markov chain makes the assumption that each event is conditionally dependent on only the previous $$n$$ events. Any events before this are assumed to have zero or negligible effect. The assumption essentially boils down to the following equality.

$$
P(x_m|x_1,\ldots,x_{m-1}) = P(x_m|x_{m-1},\ldots,x_{m-n})
$$

The most simple case is where each event is only dependent on the previous event. This leads to the following assumption.

$$
P(x_m|x_1,\ldots,x_{m-1}) = P(x_m|x_{m-1})
$$

For now I will give an explanation for generating random sequences of words for $$n=1$$. It will become clear later how things will work when $$n$$ has values of greater than 1. 

#### How does this relate to generating random text?
With a training example one can collect a sequence of transitions for each word in the training set. One can then construct a sequence of words based on the transitions in the training example. A transition being defined as the mapping $$w_1 \rightarrow w_2$$ where $$w_1$$ and $$w_2$$ are words and $$w_1$$ appears directly before $$w_2$$. Each word $$w$$, will have a corresponding list that contains every transition where $$w$$ is on the left side.

Below are some transition lists for the following test example.

>"we know what we are but know not what we may be"

{% highlight python %}
transitions["we"] = ["know", "are", "may"] 
transitions["know"] = ["what", "not"]
transitions["what"] = ["we", "we"]
transitions["are"] = ["but"]
transitions["but"] = ["know"]
transitions["not"] = ["what"]
transitions["may"] = ["be"]
{% endhighlight %}

To find the next word in the sequence one can merely chooses a random element from the current word's transition list.
{% highlight python %}
next_word(w) = random_element(transitions[w])
{% endhighlight %}

In this way one can a random sequence of words based on some training data. Obviously with a such a small training example the variation in possible sequences is very low. Usually this technique is used in conjunction with large documents or corpuses used as the training data.

To apply this in cases where the value of $$n$$ is greater than 1, the transition lists would based on sequences of $$n$$ items. So for $$n = 2$$ the following transition lists would be generated.

{% highlight python %}
transitions["we", "know"] = ["what"] 
transitions["know", "what"] = ["we"]
transitions["what", "we"] = ["are"]
transitions["are", "but"] = ["know"]
transitions["but", "know"] = ["not"]
transitions["know", "not"] = ["what"]
transitions["not", "we"] = ["may"]
transitions["we", "may"] = ["be"]
{% endhighlight %}

A the next_word function would have to be slightly adapted to accomodate this but the principle is largely the same.

In the next blog post I will show how this logic can be implemented in the [Clojure](http://www.clojure.org).
