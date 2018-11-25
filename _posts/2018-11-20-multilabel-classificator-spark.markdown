---
layout: default
title:  "Building a multi label classificator with Spark ML"
date:   2018-11-20 22:44:29 +0100
categories: Apache Spark
---

<!--ts-->
- [Introduction](#Introduction)
- [Multilabel classification](#Multilabel classification)
- [Getting the data](#Getting the data)
- [Spark ml class hierarchy](#Spark ml class hierarchy)
  * [Estimator vs Classificator](#Estimator vs Classificator)
<!--te-->

# Introduction

As part of my self-learning adventure on Apache Spark, I decided that it was time to stop doing online courses and toy projects and really get my hands dirty. So I decided to build a end-to-end ML pipeline, with real world data.

After looking a bit for public datasets to use, I found that stackoverflow periodically makes its data available through the [stack exchange data dump](https://archive.org/details/stackexchange). It met all my requirements, a lot of data to work with  and as a regular stackoverflow user, a source of interesting information / insights.

So I already had the data, the only thing left was to decided what to do with it. To build a classificator to automaticly label the questions seemed like an interesting problem to solve:
  * It gave me the opportunity to get experience on the full pipeline of machine learning with Spark, from ETL to model serving
  * A well defined problem, with a lot of labeled examples to learn from
  * It had a real / util application

The problem statement is very clear. Train a classifier on the StackOverflow posts dataset, that given a question, is able to predict the tags that apply to it. Each post has a lot of fields (you can take a lookt at this [marvelous post from stackexchange forums](https://meta.stackexchange.com/questions/2677/database-schema-documentation-for-the-public-data-dump-and-sede)). But for our problem the only relevant fields are:

* `Id`
* `PostTypeId1`
  - 1 = Question
  - 2 = Answer
  - 3 = Orphaned tag wiki
  - 4 = Tag wiki excerpt
  - 5 = Tag wiki
  - 6 = Moderator nomination
  - 7 = "Wiki placeholder"
  - 8 = Privilege wiki
* `Body`
* `Title` 
* `Tags` 

# Multilabel classification

If you ever visited StackOverflow, you will probably have seen that a question can have multiple tags (Ej. javascript and jquery). So our classifier output will have to a be a sets of tags, all those that apply to the question. 

This is known as multilabel classification, a problem in which we have multiple classes and each instance may be assigned various of those classes. There are two main approaches to solve multilabel classification problems:

* `Problem transformation methods`: Aim at transforming the problem into other which we know how to solve. Two of the most used techniques are:
  - **Binary relevance**: This is the most straitghforward approach, the idea is to independently train one binary classifier for each label. Given an unseen sample, the combined model then predicts all labels for this sample for which the respective classifiers predict a positive result
  - **Label powerset**: This approach transforms the problem into a multiclass classificaiton one, by training a classifier on each combination of labels. 
* `Algorithm adaptations methods`: Algorithms/models that have been adapted to the multi-label task, without requiring problem transformations

As to date, Spark ML doesn't offer any algorithm adapted to multilabel classification, so we will have to go with the problem transformation approach. As we will see in the following section the number of unique combination of tags is too high (tens of thousands) to train a classifier on each one, so we are only left with the binary relevance technique. 

# Getting the data

# Spark ml class hierarchy

In the binary relevance technique that I decided to adopt, you have to build a binary classifier for each class and then aggregate the results of all of them. Spark provides models for binary classification, however does not provide anything relating the aggregation of models. So you will have to build your own. 

If you take a look a look at the available classification models, you will see one named OneVsRest. This classifier implements the one vs rest approach for multiclass classification problems, where you train a binary classifier for each label. Sounds familiar? In fact the one vs rest (or one vs all) and binary relevance techniques are very much alike, though not the same (see [this excelent post](https://www.quora.com/Is-binary-relevance-same-as-one-vs-all) for and explanation on the differences).

So instead of writing by own multilabel classificator from scratch, I will just adapt the existing OneVsRest implementation to work for the multilabel case (it can be seen as a generalization of the model)

Before diving into the code, lets take a look at spark ml class hierarchy, so we can understand better the actual implementation.

![SparkML classes tree](/assets/posts/multilabel-classification-spark/sparkml_classes_tree.png)

The diagram above depicts the backbone classes of the ML library, on which most of the others build on. The diagram is oriented towards the `DeveloperAPI`, and so only shows the methods that you need to implement if you extend / implement the class. One important thing to note here is that `OneVsRest` classifier implements directly the `Estimator` class instead the `Classifier` one. We will discuss this in the following section.

### Estimator vs Classifier

The first approach I took for building the multilabel classficator, was to try to write it from scratch by implementing the `Classifier` interface. It made sense to me, since after all I wanted to write a classifier.









Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
