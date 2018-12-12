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

The diagram above depicts the backbone classes of the ML library, on which most of the others build on. The diagram is oriented towards the `DeveloperAPI`, and so only shows the methods that you need to implement if you extend / implement the class. You can see two branches, the one deriving from `Estimator` and the one deriving from `Transformer`, with a lot of similarities (in terms of structure and naming). This is because Spark separates the training and evaluation / test / exploitation stages. So, when you train a `Classifier` on your data, what you obtain is a `ClassificationModel`. This can become more clear if we take a look at the actual implementation.

{% highlight scala %}
// Code from org.apache.spark.ml.Predictor.scala, simplified for the sake of clarity
abstract class Predictor[
    FeaturesType,
    Learner <: Predictor[FeaturesType, Learner, M],
    M <: PredictionModel[FeaturesType, M]]
  extends Estimator[M] with PredictorParams {

   /**
    * Train a model using the given dataset and parameters.
    * See how the output of this model is of type M, which if defined to be a subclass of PredictionModel
    */
    protected def train(dataset: Dataset[_]): M

    ...
}
{% endhighlight %}

This implies that if we want to build or own classifer, we will need to implement both the `Estimator` and `Transformer` interfaces. This requirement will greatly influence our solution, as we will see in the next section.

### Estimator vs Classifier

If you take another look at the above diagram, you will note that `OneVsRest` classifier implements directly the `Estimator` class instead the `Classifier` one, despite being a classification technique and being in the classification package. This is mainly due to Spark ML classes being designed to implement actual concrete machine learning methods, rather than meta algorithms (or ensembles) as it is the case. 
The problem arises from the implementation of the `ClassificationModel` abstract class, rather than the `Classifier`. If you recall the diagram, this class requires us to implement the `predictRaw` method. The docstring for this method states the following:

```
Raw prediction for each possible label. The meaning of a "raw" prediction may vary between algorithms, but it intuitively gives a measure of confidence in each possible label (where larger = more confident).
```

In the case of binary classifiers (as the ones used by `OneVsRest` and binary relevance), this output if often the probability of the instance to be of class C. So if we want to implement this method, we need access to the underlying probabilities predicted by each model. Something like the following:


{% highlight scala %}
  override protected def predictRaw(features: Vector): Vector = {
    // Models is an array of binary classifiers for each class
    // We just pick the positive probability for each model ( p = P(C|X) instead of 1 - p)
    Vectors.dense(models.map{ _.predictRaw(features).toDense.values(1) })
  }
{% endhighlight %}

Unfortunatly the method predictRaw is protected, so it is not accesible outside the spark ml library making it impossible to implement this method. That's the reason I would extend the Estimator / Model interface instead.


