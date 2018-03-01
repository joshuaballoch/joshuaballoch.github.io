---
layout: post
title: Custom Packages with AWS Lambda
description: "Add support for extra python packages your Lambda function in a few simple steps."
modified: 2018-02-27
tags: [python, lambda, aws-lambda, aws, python3, serverless]
comments: true
subscribe_cta: true
enable_scroll_tracking: true
share: true
---

In the [previous post](/up-and-running-lambda) of [this series](/production-ready-aws-lambda) we built a basic setup for developing python Lambda functions. The python environment for a standard Lambda function already includes a [enough packages](https://gist.github.com/gene1wood/4a052f39490fae00e0c3) to build simple functionality. However, the default environment is by no means exhaustive - users building data analysis tools, for example, may be surprised to find common packages like Numpy missing.

The good news is that AWS makes it quite easy to deploy custom python packages with your functions. On top of that, Serverless abstracts away this dirty work, requiring just a few modifications to a project's configuration to add them in.

As a note, this post will mostly mirror the same content in [serverless' post on adding custom python packages](https://serverless.com/blog/serverless-python-packaging/), but I add this on top of the setup I introduced in the last post, and also add some discussion on deployment considerations with custom packages.

This post uses as its starting point the code built in the [previous post](/up-and-running-lambda) in this series, [which you can download here.](https://github.com/joshuaballoch/deploying-lambda/tree/399de54dc9b0d5ce7cdd8298abe91a27f084971f
)

## `import` to break our function

As in the [last post](/up-and-running-lambda), we won't build any real functionality into our function. We will just add any old library via an `import` statement, see that our function is broken, and then fix our setup. For consistency, let's add `numpy`, just like [serverless did in their post](https://serverless.com/blog/serverless-python-packaging/). 

Add an `import numpy` to our `handler.py` file:

```
import numpy

def call(event, context):
    print("hello world!")
```

If we try loading our handler function now, it will fail. Feel free to deploy to AWS and see the function fail there too.

## Adding Requirements

Our first step in fixing our function is to add numpy to the `requirements/requirements.in` file. It was empty before, so now file will just be one line long:

```
numpy
```

Let's install this package in our local development environment:

```
$ pip-compile --output-file requirements/requirements-dev.txt requirements/requirements-dev.in
$ pip install -r requirements/requirements-dev.txt
```

Now we can load and call our handler locally without issue, but it still won't run properly on AWS.

## Prepare the deploy

To deploy custom python packages, we can use the `serverless-python-requirements` npm package. Roughly speaking, when Serverless deploys a function to AWS, it is just zipping up our project folder and uploading it. This npm package adds functionality to this process to compile and include in that upload the python packages we want at runtime.

Create a `package.json` file to specify this as a development dependency, and then run `npm install`.

```
{
  "name": "deployable-lambda",
  "description": "",
  "version": "0.1.0",
  "dependencies": {},
  "devDependencies": {
    "serverless-python-requirements": "^3.0.10"
  }
}
```

Let's exclude `node_modules` from the deployed package by modifying our `serverless.yml` config. Users using `git` may also wish to add it to `.gitignore`.

```
package:
  exclude:
    ...
    - node_modules/**
```

Now we'll need to add some options to the `serverless.yml` config to tell serverless that we want to deploy custom python packages:

```
plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: non-linux
```

With this in place, serverless will attempt to build a deployment package using Docker. It uses docker to compile required python packages in an environment comparable to the one AWS runs your function within. If we were to compile them directly on Mac or another non-linux OS, we would risk running into issues where the two systems differ.

## Installing Docker Machine

To proceed with the deployment, we'll now need to install docker-machine.

Mac users can install this using `brew`:

```
$ brew install docker-machine
```

Once it is installed, we need to create a main machine that will be used for the builds:

```
$ docker-machine create main
```

We will need to start it so that serverless can communicate with it for deployments:

```
$ docker-machine start main
$ eval $(docker-machine env main)
```

## Deploying

Now that we have the right dependencies installed and a docker machine running, we can run our first deploy with a custom python package.

The first thing we'll need to do is use `pip-compile` to create a `requirements.txt` file in the main project directory. Serverless' python-requirements package looks here to know what custom packages to deploy. Note that in this case, we're outputting just based on the `requirements/requirements.in` file, not the development one.

```
$ pip-compile --output-file requirements.txt requirements/requirements.in
```

Now we're ready to deploy.

```
$ serverless deploy --stage staging
```

## Considerations

This setup allows for an easy way to deploy Lambda functions with custom python packages. That doesn't come without a cost though. For one thing, our development environment now depends on a new npm package, and docker.

More tangibly, the deployment package gets a lot larger. With just numpy installed, the new deployment package is 21 MB in size, up from 31 KB earlier. To understand more deeply, it can be informative to check out what serverless actually uploads to AWS. The last deployment is cached locally inside `.serverless/hello-world.zip`.

While this size isn't of itself such a big deal, between this and the docker build, deploy times slow down considerably. Also, as users pay (minor) bandwidth charges on uploads to S3 and Lambda, it is worth noting that this may incur some increased charges.

Aside from that, AWS imposes a limit on the size of the final, deployed Lambda functions it will run. This limit (250 MB) is small enough that some users might run up against it after adding in a few packages.

To help deal with this limit, it's possible to compress dependencies in the deployed package. This approach will keep the deployed code for your custom python packages compressed, even at runtime.

Two additions are needed to compress dependencies:

1) An addition to the serverless.yml config:

```
custom:
  pythonRequirements:
    zip: true
    ...
```

2) An addition at the top of any lambda function:

```
try:
    import unzip_requirements
except ImportError:
    pass
```

Note that this doesn't make a sizeable dent in the package upload size (21.45 MB to 21.07 MB) which makes sense, as the upload was already being zipped. However, it will allow users' deployed functions to include more libraries, as the 250 MB limit is on the deployed package and its dependencies at runtime, not the package size on upload.

## End

That's it, I think, for deploying custom packages with AWS Lambda functions. For those interested, here are some links to the code produced in this tutorial:

- [Basic Setup, including Custom Py Packages, unzipped](https://github.com/joshuaballoch/deploying-lambda/tree/custom-py-packages)
- [Basic Setup, including Custom Py Packages, zipped](https://github.com/joshuaballoch/deploying-lambda/tree/custom-py-zipped)

