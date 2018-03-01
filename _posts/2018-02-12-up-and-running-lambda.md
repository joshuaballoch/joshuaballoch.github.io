---
layout: post
title: Up and Running in AWS Lambda
description: "Get a development environment set up for Python Lambda functions, with one-line deploys and staged environments."
modified: 2018-02-12
tags: [python, lambda, aws-lambda, aws, python3, serverless]
comments: true
subscribe_cta: true
enable_scroll_tracking: true
share: true
---

This post is the first in a [series](/production-ready-aws-lambda) about getting a production-ready setup going for AWS Lambda using Python as the runtime.

In this post, we'll focus on getting up and running with a Lambda function, with the ability to deploy using a single command. We'll also get the local development environment set up, and build space into the configuration for deployed to staged environments.

Just to note - the end goal of this post is a lambda function that won't actually do anything. In future posts, we'll add functionality that simulates doing data processing in a data pipeline. We'll develop and test that function locally before deploying. For now, we'll just build the base dev environment and deploy an empty function to AWS>

## Getting Started

We'll be building Python Lambda functions, and deploying them using a tool called [Serverless.](https://serverless.com/) `Serverless` is a toolkit that provides a command-line interface for deploying and managing serverless functions. It works with AWS Lambda, and also supports other cloud providers' serverless implementations.

Before we get started, make sure you have serverless installed, or [head over to serverless' installation instructions.](https://serverless.com/framework/docs/getting-started/)

Let's create our starting function using serverless' template for a Python 3.x runtime:

```
$ mkdir deployable-lambda && cd deployable lambda
$ serverless create --template aws-python3
```

Serverless' initial template is quite barebones, and includes just a single `handler.py` file - the "lambda function", and a `serverless.yml` file for specifying your service's configuration.

Let's update the `handler.py` function to rename the main method to `call`, and have it simply print a line to the log when executed.

```py
def call(event, context):
    print("hello world!")
```

Inside of `serverless.yml`, let's get rid of the extra comments. Then we need to modify the config to reflect our method renaming. Let's name our service "hello-world", and our first function "hello_world"

```
service: hello-world

provider:
  name: aws
  runtime: python3.6

functions:
  hello_world:
    handler: handler.call
```

## Staged Environments

Serverless allows for an easy way to deploy any given service to multiple, staged environments through its `stage` option. Let's modify the config file to accept a command line option `stage` for this:

```
provider:
    name: aws
    runtime: python3.6
    stage: ${opt:stage, 'dev'}
```

Now when we deploy our function, we can specify which environment we're deploying to via `serverless deploy --stage [staging|prod]`

## Development Environment

Our aim is to build a development setup for Lambda functions, writing and testing python code locally before deploying it to AWS. For this, it would be great to have a predictable local dev environment. Luckily, python's `virtualenv` does exactly this; we'll use it to create a python environment specifically for our project.

If you haven't already installed python 3.x, go ahead and install it with `brew`. Then create a virtualenv configuration in `.venv` of the project directory:

```
$ virtualenv .venv -p python3
```

Activate this environment:

```
$ source .venv/bin/activate
```

In my experience with Lambda functions, I have often seen the need for different packages installed specifically for local development that we won't actually need in the deployed environment. With this in mind, we'll create a way for managing the deployed and development environments separately using `pip-tools`.

Install `pip-tools`:

```
$ pip install pip-tools
```

Let's create a `requirements/` directory and create a `requirements.in` file for our deployed runtime. Right now, this will be empty.

Now let's create a `requirements/requirements-dev.in` file. In this file, we'll specify python packages that we want in the local environment that we don't want in the deployed environment. At the top of this file, we'll recursively install any requirements of the main environment. We can also list any development-specific packages below.

```
-r requirements.in

# Other packages get listed here, e.g. pytest==3.4.0
```

With this development setup, we'll be able to add packages to the local development environment by:

```
$ pip-compile --output-file requirements/requirements-dev.txt requirements/requirements-dev.in
$ pip install -r requirements/requirements-dev.txt
```

## Preparing to deploy

Let's get our function set up for its first deploy. 

When serverless runs a deployment, it will by default include all the files inside of the project directory. We want to keep our deployment package lean, so let's instruct serverless to ignore certain files and directories, below the `provider` block.

```
package:
  exclude:
    - .venv/**
    - .git/**
    - __pycache__/** # for python 3.x
    - '*.pyc' # if using python 2.x

```

As an aside, if you're using `git` for version control, you can go ahead and add the following to your gitignore:

```
.venv/
*requirements*.txt
```

Now we're ready for the first deploy. Note if you haven't already placed your AWS credentials into `~/.aws/credentials`, check out [the instructions for that here.](https://docs.aws.amazon.com/cli/latest/userguide/cli-config-files.html) With that sorted, go ahead and run:

```
serverless deploy --stage staging
```

Whenever you're ready to deploy new changes to a serverless function, that single command will handle it all.

Removing a service is as easy as deploying:

```
serverless remove --staging staging
```

We now have a simple starting lambda function to build off of, we can deploy it with a single command to multiple environments, and we have our local development environment set up.

As a side note, many people will run into the need to use custom python packages in their Lambda functions. Serverless has good support for this use case, but the setup required for this is a little more involved, and is the subject of [the next post of this series](/custom-packages-lambda).

The code created in this post is [available here.](https://github.com/joshuaballoch/deploying-lambda)
