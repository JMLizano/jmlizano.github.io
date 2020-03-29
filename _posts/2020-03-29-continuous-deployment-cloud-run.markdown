---
layout: default
title:  "Continuous deployment to cloud run"
date:   2020-03-29 22:44:29 +0100
categories: Google Cloud
---

<!--ts-->
- [Introduction](#Introduction)
- [Github actions vs Cloud Build](#Getting the data)
<!--te-->


# Introduction

In this post I will talk about setting up a 
[Continous integration and continuous deployment](https://codeship.com/continuous-integration-essentials)(CI/CD) 
pipeline, for a web application running on Google Cloud Run and hosted on Github.
CI/CD has become quite popular in recent times. While continuous integration is
quite extended, and easier to set up, is a bit more difficult to get the oportunity
to set up continous delivery. 

Since I had been waiting for a while already to get into some project where I could set up such
a workflow, I decided to test it in [one of my personal projects](). The flow that I want to
achieve is depicted in the following figure. 

![ci/cd workflow](/assets/posts/ci-cd-cloud-run/cicd-workflow.png)

# Build & deployment tools

The first decision is which CI/CD tool to use. This can be harder than you may think at first,
due to the miriad of available options nowadays: Jenkins, CircleCI, CodeShip, Github Actions,
Travis CI...And those are only the first that came to my mind. As with any other technology
choice, the first thing that you need to define are your requirements. 

You should think both about technical (supported languages, local execution, ease of
implementaton, etc.) and non technical (cost, community support, documentation, etc.)
requirements. In my case I aimed at the following:

* A fully managed solution, ideally with few to none setup required.
* As flat as possible learning curve, I don't want to spend hours digging around documentation 
and domain specific concepts. 
* Free to use (at least for my use case).
* Ability to execute in my laptop. I had hard times in the past debugging this kind of tools due 
to not being able to execute the pipeline in my local environment.

With all this in mind, the recently released [Github Actions](https://help.github.com/en/actions)
seemed like the perfect fit. It is a tool offered by Github itself, so the integration is seamles,
and does not require opening a new account in a third party service. It is also a managed service
(although it offers the possibility to host it yourself), so does not require to set up any extra
server. Finally although it doesn't offer an official supported way to execute the pipelines in
your local environment, there is an [open source tool](https://github.com/nektos/act) for it.

Since I am deploying to Cloud Run, there is one last decision to make. Google cloud offers it's 
own tool for continuous delivery, [cloud build](https://cloud.google.com/cloud-build). This tool
allows to connect it to some source control repository, and define events to trigger the
deployment. For example you can trigger a new build every time there is a new tag pushed to the
master branch of your GitHub repository.
But it doesn't provide continuous integration on it's own, so if you want to use cloud build, 
you need to break your workflow in two:

* Github Actions (or whatever tool you want to use) for continous integration
* Cloud Build for continous delivery, based on triggers from Github. 

S, is either use Github Actions for CI/CD or use it only for CI and use Cloud Build for the 
deployment. There are pros and cons for each of these approaches:

* Github Actions for both CI & CD.
    * Only one place for configurations
    * Only one syntax to learn
    * More fine grained control over when to trigger the workflow

* CI happens in Github and CD in Google cloud Build
    * Feels right to use Github for CI, since Github focus is on code
    itself and Google Cloud for CD, since Google focus is on deployment.
    * Official way for running locally
    * You don't have to share any deployment keys (GKE, gcloud shell, etc.)
    with Github
    * Official documentation & support on the deployment part

For the me, the convenience of only having to learn one syntax, and have all the configuration
in the same place outweights the rest. So I will go with Github Actions. 

# Configuring Github Actions

## Continous integration

The only thing you need to set up Github Actions, is an action configuration file under the
`.github/workflows` folder. Github offers [examples of these configuration files](https://github.com/actions/starter-workflows) 
for almost all languages that you may use.

A basic configuration has the following aspect:

{% highlight yaml %}
name: Test pull request

on:
  pull_request:
    branches: [ master ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:

    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    
    - name: Run tests
      run: sbt test

{% endhighlight %}

As you can see, the file is formatted as YAML. There are two main sections here:

* `on`: Specifies when to trigger this workflow. You can set multiple events as triggers,
  Github offers [detailed documentation on the available events](https://help.github.com/en/actions/reference/events-that-trigger-workflows).
  In our example, we have set the workflow to trigger on pull requests to the `master` branch. 

* `jobs`: This is the actual list of jobs that we want to run as part of this workflow. Each job
  has the following properties:
    * `runs-on`: Specifies where do you want to run your workflow. Github offers a set of
      [pre-built virtual images](https://help.github.com/es/actions/reference/virtual-environments-for-github-hosted-runners)
      that you can use. You can also [host your own runner](https://help.github.com/en/actions/hosting-your-own-runners)
      if you want. In our case I have selected the `ubuntu-latest` runner offered by Github, that
      includes [**a lot** of build tools](https://github.com/actions/virtual-environments/blob/master/images/linux/Ubuntu1804-README.md)
      already.
    * `steps`: List of actions (tasks) to execute as part of this job. Github offers quite a lot,
      of actions and there are also a lot more built by the community, so most likely you will 
      find whatever you need. But if that is not the case, you can always 
      [build your own action](https://help.github.com/en/actions/building-actions).
  
  In our workflow we only have one job, `test`, where we first checkout the repository, then 
  set up JDK 1.8 as the Java version to use (since there are several ones installed on the 
  image) and finally execute the tests for our Scala project.

This setup, as simple as it is, will work out of the box for most simple projects. In my case,
there is still one missing step, since my project depends on the existence of some environment
variables. Fortunately, Github offers and [easy way to set environment variables](https://help.github.com/en/actions/configuring-and-managing-workflows/using-environment-variables)
in the workflow.
In case your variables hold some sensitive information (like it is my case), it is also possible
to [create secrets in Github](https://help.github.com/es/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets),
and make them available to the workflow as environment variables.
After adding this variables, the end file is as follows.

{% highlight yaml %}
name: PR test CI

on:
  pull_request:
    branches: [ master ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:

    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    
    - name: Run tests
      run: sbt test
      env:
        TWITTER_CONSUMER_TOKEN_KEY: ${{ secrets.TWITTER_CONSUMER_TOKEN_KEY }}
        TWITTER_CONSUMER_TOKEN_SECRET: ${{ secrets.TWITTER_CONSUMER_TOKEN_SECRET }}
        TWITTER_ACCESS_TOKEN_KEY: ${{ secrets.TWITTER_ACCESS_TOKEN_KEY }}
        TWITTER_ACCESS_TOKEN_SECRET: ${{ secrets.TWITTER_ACCESS_TOKEN_SECRET }}
{% endhighlight %}

And that's it, this is all you need to setup a basic CI flow, where the project tests will be
executed each time a pull request agains `master` is created. Of course, you can futher improve
this flow:

    * Require that the [tests pass to be able to merge](https://help.github.com/en/github/administering-a-repository/defining-the-mergeability-of-pull-requests)
    * Or even some kind of automating merging after tests pass

But for me this simple setup is enough.

## Continous deployment

Now that we have 'CI' in place, we are still missing the 'CD' part. For that the first step is
to write the code the perform the actual deployment. In my case I have written this simple bash
script:

{% highlight bash %}
#!/bin/bash -xue

PROJECT=myproject
IMAGE_NAME=diditweetthat
VERSION=$(sbt 'inspect actual version' | grep "Setting: java.lang.String" | cut -d '=' -f2 | tr -d ' ')

gcloud config set project ${PROJECT}

# Build & publish image
docker build  --build-arg VERSION=${VERSION} --target prod -t ${IMAGE_NAME}:$VERSION .
docker tag ${IMAGE_NAME}:$VERSION eu.gcr.io/${PROJECT}/${IMAGE_NAME}:$VERSION
docker push eu.gcr.io/${PROJECT}/${IMAGE_NAME}:$VERSION

## Deploy to cloud run
source .env || true # This file contains all required secrets
APPLICATION_SECRET=$(head -c 32 /dev/urandom | base64)

gcloud run deploy diditweethat \
  --image eu.gcr.io/${PROJECT}/${IMAGE_NAME}:$VERSION \
  --platform managed \
  --max-instances=2 \
  --memory=2Gi \
  --cpu=1 \
  --port=9000 \
  --set-env-vars=VERSION=${VERSION},TWITTER_CONSUMER_TOKEN_KEY=${TWITTER_CONSUMER_TOKEN_KEY},TWITTER_CONSUMER_TOKEN_SECRET=${TWITTER_CONSUMER_TOKEN_SECRET},TWITTER_ACCESS_TOKEN_KEY=${TWITTER_ACCESS_TOKEN_KEY},TWITTER_ACCESS_TOKEN_SECRET=${TWITTER_ACCESS_TOKEN_SECRET},APPLICATION_SECRET=${APPLICATION_SECRET} \
  --allow-unauthenticated \
  --region=europe-west1
{% endhighlight %}

Once we have this, we only need to create a new workflow, that will execute this script on 
`push` events to the master branch (so it will be executed after merging a pull request). This
workflow is quite similar to the previous one (since we still want to exectute the tests, just
in case), with a couple of extra steps to perform the deployment. 

{% highlight yaml %}
name: Deploy

on:
  push:
    branches: [ master ]

jobs:
  test-build-deploy:
    runs-on: ubuntu-latest
    steps:

    - name: Checkout
      uses: actions/checkout@v2

    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    
    - name: Run tests
      run: sbt test
      env:
        TWITTER_CONSUMER_TOKEN_KEY: ${{ secrets.TWITTER_CONSUMER_TOKEN_KEY }}
        TWITTER_CONSUMER_TOKEN_SECRET: ${{ secrets.TWITTER_CONSUMER_TOKEN_SECRET }}
        TWITTER_ACCESS_TOKEN_KEY: ${{ secrets.TWITTER_ACCESS_TOKEN_KEY }}
        TWITTER_ACCESS_TOKEN_SECRET: ${{ secrets.TWITTER_ACCESS_TOKEN_SECRET }}
    
    - name: Setup gcloud command line
      uses: GoogleCloudPlatform/github-actions/setup-gcloud@master
      with:
        version: '286.0.0'
        service_account_email: ${{ secrets.SERVICE_ACCOUNT_EMAIL }}
        service_account_key: ${{ secrets.SERVICE_ACCOUNT_KEY }}
        export_default_credentials: true
    
    - name: Setup docker authentication to Google Cloud
      run: gcloud auth configure-docker
    
    - name: Build docker image and deploy to cloud run
      run:  bash cloud_run_deploy.sh
{% endhighlight %}

The extra steps are basically for setting up the `gcloud` command line tool, the docker
authentication to google cloud registry, and finally executing our previous script. There
is an important detail here, that you may have noticed. The `service_account_email` and
`service_account_key` variables in the step that set ups `gcloud` reference some secrets
in my Github account.

When you interact with Google Cloud, you need to be authenticated. In you laptop, when 
you are the one issuing commands against the Google Cloud API, you are using (normally)
your Google account you authenticate yourself.
But in this case, it is the Github agent the one issuing commands agains Google Cloud API,
so it needs some credentials to be able to authenticate. For this use case, Google created
[service accounts](https://cloud.google.com/iam/docs/service-accounts). A special type of
accounts intended for automation purposes.

So before being able to execute the above workflow, you will have to create a service 
account with the appropriate permissions. For that, you can use the following script:

{% highlight bash%}
#! /bin/bash

PROJECT_ID=myproject
NAME=diditweetthat

gcloud config set project $PROJECT_ID
# Create service account
gcloud iam service-accounts create $NAME
# Associate the owner role to the service account
gcloud projects add-iam-policy-binding \
     $PROJECT_ID \
     --member "serviceAccount:$NAME@$PROJECT_ID.iam.gserviceaccount.com" \
     --role "roles/owner"
# Create a JSON key associated to that account, that we can use
# to authenticate against Google Cloud.
gcloud iam service-accounts keys create \
    ${NAME}.json \
    --iam-account $NAME@$PROJECT_ID.iam.gserviceaccount.com
{% endhighlight %}

**Please note that the account created with this script has the owner role associated with it.
This role provides basically admin access to Google Cloud. If you intend to deploy this in a real
production environment, make sure to assign a more restricted set of permissions**

The last step of this script is creating the key that we need to provide to the `setup-gcloud`
action in the above workflow. It is worth noting that the key is the whole JSON itself, not only
the `private key` field inside the JSON. The only thing that you need to do is to encode the JSON
as base64, and use that as the key value.

{% highlight bash%}
$ cat diditweetthat.json | base64
{% endhighlight%}

For the `service_account_email`, use the value of `client_email` field inside the JSON. Once you do
this, you should be able to execute the workflow.

To test it you can just create a dummy pull request. You will see how the test workflow is executed
as soon as you create the pull request. And once you merge it, the deploy workflow will be triggered.


# Conclusion

Github Actions has really lowered the bar for entering the 'CI/CD' playground. Even if this is not a
very sophisticated workflow, is more than enough for small projects, or as the first steps in this
area. And it took no more than a few hours to set everything up! And that's taking into account that
this was my first interaction with Github Actions, so I have to spent some time in understanding all
its concepts.
