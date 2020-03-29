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
- [OneVsRest](#OneVsRest)
  * [Estimator](#Estimator)
  * [Transformer](#Transformer)
  * [Some extra notes](#Some extra notes)
- [Serialization](#Serialization)
- [Serving](#Serving)
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

You can find all the code related to this post in [Github](git@github.com:JMLizano/StackOverflow-Question-Tagging.git)

# Multilabel classification

If you ever visited StackOverflow, you will probably have seen that a question can have multiple tags (Ej. javascript and jquery). So our classifier output will have to a be a sets of tags, all those that apply to the question. 

This is known as multilabel classification, a problem in which we have multiple classes and each instance may be assigned various of those classes. There are two main approaches to solve multilabel classification problems:

* `Problem transformation methods`: Aim at transforming the problem into other which we know how to solve. Two of the most used techniques are:
  - **Binary relevance**: This is the most straitghforward approach, the idea is to independently train one binary classifier for each label. Given an unseen sample, the combined model then predicts all labels for this sample for which the respective classifiers predict a positive result
  - **Label powerset**: This approach transforms the problem into a multiclass classificaiton one, by training a classifier on each combination of labels. 
* `Algorithm adaptations methods`: Algorithms/models that have been adapted to the multi-label task, without requiring problem transformations

As to date, Spark ML doesn't offer any algorithm adapted to multilabel classification, so we will have to go with the problem transformation approach. As we will see in the following section the number of unique combination of tags is too high (tens of thousands) to train a classifier on each one, so we are only left with the binary relevance technique. 

# Getting the data

The first step of any machine learning project is getting the data, and prepare 
it to make work with it easier. If you download the data from the above link, you 
will see that there is a huge XML file for each entity (Ex. Post, User, ect.).

Unfortunatly, there is not an entity for questions, they are included in the `Posts.xml`
file. This can be solved easily, since each post has a `PostTypeId` field, to 
indicate the type of the post. All questions have `PostTypeId = 1`.

Aside from this, this format has some other inconvenients:

* Reading XML files is very slow
* The files are not partitioned, so we can not take advantage of the Spark
parallelization capabilities.

So before start any work on this data, we will extract the questions and transform it to date-partitioned 
[parquet files](parquetreference). Parquet is a columnar format, that is designed with 
parallelization and speed in mind.

To transform the data from XML to parquet, we will use Spark itself, and the 
[spark-xml](reference) library developed by databricks. Conceptually, this
program follows the following structure.

```scala
val questions = spark
                  .read
                  .format("com.databricks.spark.xml")
                  .option("rowTag", "row")
                  .where("_PostTypeId = 1 AND _Tags is not null AND _CreationDate is not null")
```

There are two problems with this approach:

* It is slow, due to the file not being partitioned
* There is a catch with the `spark-xml` library, it will not correctly
process XML entitied with no inner childs.

To solve this problems I created a [bash script](reference) that split the XML file in chunks of
a specified number of lines, and adds a dummy child to each `row` object. After this I was
finally able to read the XML, extract the questions and write them as parquet files using
Spark. You can see [here](reference) see the final code.


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

This implies that if we want to build or own classifer, we will need to implement both the 
`Estimator` and `Transformer` interfaces. This requirement will greatly influence our solution, 
as we will see in the next section.

## Estimator vs Classifier

If you take another look at the above diagram, you will note that `OneVsRest` classifier 
implements directly the `Estimator` class instead the `Classifier` one, despite being a 
classification technique and being in the classification package. This is mainly due to 
that in the beginnning of the Spark ML library, classes were designed to implement actual 
concrete machine learning methods, rather than meta algorithms (or ensembles) as it is the case. 

This has changed, and now meta algorithms are better supported. In fact, there are already 
[plans to refactor OneVsRest to extend the Classifier interface in Spark 3.0](https://issues.apache.org/jira/browse/SPARK-8799)

I will follow the current implementation and extend the `Estimator` class also. For two reasons,
so we can use the actual `OneVsRest` implementation, and because `ClassificationModel` methods 
signature don't fit what we need. The `ClassificationModel`  implements two methods to get the 
actual predictions, `raw2prediction` and `predict`. If we take a look at the signature of these 
methods, we can see that they expect a `Double` as output. But what we want to return is an 
`Array[Double]` since each question can have multiple labels. Changing this would imply overriding 
almost all of the `ClassificationModel` methods, killing the purpose of extending it.

{% highlight scala %}
  /**
   * Predict label for the given features.
   * This method is used to implement `transform()` and output [[predictionCol]].
   *
   * This default implementation for classification predicts the index of the maximum value
   * from `predictRaw()`.
   */
  override def predict(features: FeaturesType): Double = {
    raw2prediction(predictRaw(features))
  }

  /**
   * Given a vector of raw predictions, select the predicted label.
   * This may be overridden to support thresholds which favor particular labels.
   * @return  predicted label
   */
  protected def raw2prediction(rawPrediction: Vector): Double = rawPrediction.argmax
{% endhighlight %}

# OneVsRest

Okay, we have decided to adapt the current `OneVsRest` implementation, So what is it missing to able 
to handle multilable problems? We have  to look at the implementation for the `Estimator` and `Transformer` interfaces mentioned above. 

## Estimator

The main method to implement when extending the `Estimator` class is the `fit` method. It is responsible for
performing all the actual training of the model. The main change I introduced, was support for label columns
of type `Array[String]`, instead of `Double`, just for convenience.

The core part of the `fit` method consists on fitting a model for each label. 

{% highlight scala %}
override def fit(dataset: Dataset[_]): OneVsRestMultiModel = {
  ...
    // create k columns, one for each binary classifier.
    val labels = getLabels(dataset)
    var labelsMapping =  Map[Int, String]()
    val models = labels.zipWithIndex.map { case (label, index) =>
      labelsMapping = labelsMapping + ((index, label))
      val labelColName = "mc2b$" + index
      // This is one of the changes with the OneVsRest implementation
      // we support here labelCol being an Array of strings
      val trainingDataset = multiclassLabeled.withColumn(
        labelColName, when(array_contains(col($(labelCol)), label), 1.0).otherwise(0.0))
      val classifier = getClassifier
      val paramMap = new ParamMap()
      paramMap.put(classifier.labelCol -> labelColName)
      paramMap.put(classifier.featuresCol -> getFeaturesCol)
      paramMap.put(classifier.predictionCol -> getPredictionCol)
      classifier.fit(trainingDataset, paramMap)
    }.toArray[ClassificationModel[_, _]]
  ...
}
{% endhighlight %}

You can see the full code [on Github](https://github.com/JMLizano/StackOverflow-Question-Tagging/blob/master/app/com/jmlizano/classification/OnevsRestMultiLabel.scala#L316).


## Transformer

For the `Transformer` the main method to implement is the `transform` method, responsible for adding a new
column to the input `DataFrame` with the computed predictions. Basically, for each row, this method collects 
the prediction of each model in a `Map`, where the key indicates the label and the value the prediction for 
that label. 
Remember that for each class we have trained a binary classifier, so 0 indicates that the label does
not apply to the row and 1 that the label applies to the row. For example in case we have only two labels: python and
javascript, assuming we enconde python as 1 and javascript as 0, the map would be something like:

Row 1 -> {0 -> 0, 1 -> 1} # The label python applies to this question
Row 2 -> {0 -> 1, 1 -> 0} # The label javascript applies to this question
etc.

The main change from the original implementation is related to this `Map`. In the original `OneVsRest` instead of 
binary predictions, they compute the 'raw prediction' (some sort of confidence), and then they choose the label 
with the highest confidence, like the following.

{% highlight scala %}
  // output the index of the classifier with highest confidence as prediction
  val labelUDF = udf { (predictions: Map[Int, Double]) =>
    predictions.maxBy(_._2)._1.toDouble
  }
  // output label and label metadata as prediction
  aggregatedDataset
    .withColumn(getPredictionCol, labelUDF(col(accColName)), labelMetadata)
{% endhighlight %}

In our case, we want to output all the labels that apply to the question, so the code looks like the following.

{% highlight scala %}
  // output the index of all classifiers with positive prediction
  val labelUDF = udf { (predictions: Map[Int, Double]) =>
    predictions.collect { case pred if pred._2 == 1 => pred._1.toDouble }.toSeq
  }
  aggregatedDataset
          .withColumn(getPredictionCol, labelUDF(col(accColName)))

{% endhighlight %}

## Some extra notes

Aside from the exposed above, there is one method that applies to both `Estimator` and `Transformer`, the
`transformSchema` method. This method needs to be implemented by all `PipelineStages` classes, and serves 
as a way of 'static' checking of the code. It should validate that the input schema is suitable for the stage 
and also produce what the expected output of your pipeline stage is based on any parameters set and an input schema.

In my implementation I have skipped the part of validating the input, since Spark does not offer any method
for checking that the type of Column is an Array, and also I can not use the default implementation, since
it is checks the label column to be numeric (at the `Predictor` level), which is not the case.

# Serialization

With the above, we have covered several steps of the machine learning pipeline, from data extraction to model training.
And along the way we have take a look at the internals of the Spark ml library. Now, 
we will dive in the final steps of productionizing a machine learning model: Serialization & Serving of the model.

Unfortunately support for serialization in the Spark ml library is not very good. In the old mllib there was
some kind of PMML support, but it still hasn't been added to the new ml package.

So the choices are either implement your own serialization layer, or use Spark's one and limit yourself to 
only being able to read the model again with Spark. Since this is only a learning project, I have used the
Spark serialization, but in real world project, you would need to implement your own serialization, in order
to be able to serve it without requiring Spark afterwards. 

There are some interesting projects that offer support in this regard, like [mlflow](https://mlflow.org/) 
and [mleap](https://github.com/combust/mleap). These projects also offer tools for the serving layer, 
making them a very interesting approach.

The implementation of the serialization itself is not very interesting. Spark offers a couple of traits
for it `MLReadable` and `MLWritable`. This part was easy, since basically the only thing I had to do is 
copying over the corresponding code from the `OneVsRest` implementation.

# Serving

For serving the model, I found again that Spark does not offer any kind of tool. And since I am using the
Spark serialization layer, I can't not use any of the above mentioned tools. So I opted for the simplest
approach, building a basic HTTP server myself to serve the model.

For the HTTP server I choose to use the [Play framework](https://www.playframework.com/). Is one of the 
most popular HTTP frameworks in Scala and it is very easy to get started with it. 

The structure of the application is very simple, is depicted at a high level in the following figure.

![SparkML classes tree](/assets/posts/multilabel-classification-spark/serving_model.png)

Basically the idea is to create an object that encapsulates the model, that we read from disk, and 
offers a simple predict method to use this model. Since objects in Scala are singletons, we don't have
to worry about avoiding creating a model object for each http connection.