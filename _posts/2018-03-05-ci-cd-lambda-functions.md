---
layout: post
title: Continuous Integration & Deployment (CI/CD) with Lambda Functions
description: "Let's add in Travis-CI to automatically build our test suite & deploy our lambda function to AWS."
modified: 2018-03-05
tags: [tdd, python, lambda, aws-lambda, aws, python3, serverless, ci, continuous-integration]
comments: true
subscribe_cta: true
enable_scroll_tracking: true
share: true
---

> Note: This is the fourth post in a [series on production-ready AWS Lamdba](/production-ready-aws-lambda)

So far in this series, we’ve built up nearly all the basic needs of a production-ready Lambda function, from the local development environment, one-line deploys, and adding in testing.

Now that we have automated tests, it would be nice to add continuous integration & deployment to our setup. With CI involved, we can display test results directly in Github Pull Requests. We can also use these kinds of tools to automate deployments to AWS.

To explore adding in these features, we'll pick up the code where we left it off [in the previous post, on testing lambda functions](/testing-lambda-functions) and get cracking. [Fork/Download that code here.](https://github.com/joshuaballoch/testing-lambda-py/tree/334f4a78ef96d6055fa56cc68df9f9c194c32c55)

# Quick word on CI/CD

[Continuous integration](https://en.wikipedia.org/wiki/Continuous_integration) & [continuous deployment](/http://searchitoperations.techtarget.com/definition/continuous-delivery-CD) are development practices that centre on the idea that a software project should be committed, reviewed, merged, built, and deployed often. What "often" means is up to choice - it can be daily, or even multiple times a day, and shouldn't be once a month. There's plenty of good reasons for CI/CD and plenty of supporting resources. If you're unfamiliar with these concepts, I recommend going and checking them out more deeply.

For the purposes of this post, let's assume we're already using Git for version control through a web-based platform like Github (or Gitlab), and that new features are built on development branches that are merged back to `master` using Pull Requests.

# Travis CI

The setup we want to build will automatically run the test suite on new Pull Requests (PRs). Our setup will also run the test suite & deploy to staging when new code gets merged to `master`. We want to use a web-based CI tool that integrates with Github, so that new PRs and merges to master directly integrate with it.

[Travis CI](http://travis-ci.org) one such solution. It is free to use for open source projects, and has a straightforward interface. We'll use it for this post, but the features covered here can be implemented in most other CI tools too. Some other popular options include [Circle CI](http://circleci.com) and Jenkins.

# Automated Builds

Let's start by getting our test suite to run builds on new PRs and on new commits to master. For Travis, the configuration of a project starts with a `travis.yml` file. In here, we'll tell Travis what software to install and commands to run with each build.

Here's a starting configuration for the project as we've built it:

```
# In .travis.yml
language: python
python:
  - "3.6"
install:
  - pip install pip-tools
  - pip-compile --output-file ~/requirements-dev.txt equirements/requirements-dev.in
  - pip install -r ~/requirements-dev.txt
script: pytest test_handler.py
```

The first lines of this config tell Travis what language to use. The `install` section lists commands Travis will run in the installation phase of each build. Here we're installing the same python packages as we do locally, using pip.

The `script` section is the meat of the build - it's here where we tell Travis to run our test suite. All this means for our project is running `pytest` with our test file.

The next step is to link up a Github repository hosting this code to Travis. Create a public Github repository for your code, if you haven't already, and log in to Travis at [https://travis-ci.org/profile/](https://travis-ci.org/profile/). On this page, you'll be prompted to "toggle on" repositories you want to build on Travis. Activate the repository hosting this code, then head to that project's Travis page (`https://travis-ci.org/<username>/<repo-name>`).

Now push the branch containing our basic `.travis.yml` file to Github. You should see a build get started on Travis. My first build with our basic config looks like this (check it out [here](https://travis-ci.org/joshuaballoch/testing-lambda-py/builds/349218884)):

```
Worker information
hostname: 5a644d69-8585-45c6-8a2e-965e0d50d7de@1.i-088e953-production-2-worker-org-ec2.travisci.net
version: v3.5.0 https://github.com/travis-ci/worker/tree/77dbc57c72d00592aeb754773b712da843c7e00d
instance: 9baef73 travisci/ci-garnet:packer-1512502276-986baf0 (via amqp)
startup: 568.486347ms
mode of ‘/usr/local/clang-5.0.0/bin’ changed from 0777 (rwxrwxrwx) to 0775 (rwxrwxr-x)
system_info
Build system information
Build language: python
Build group: stable
Build dist: trusty
# ... truncated
$ git clone --depth=50 https://github.com/joshuaballoch/testing-lambda-py.git joshuaballoch/testing-lambda-py
# ... truncated
$ cd joshuaballoch/testing-lambda-py
0.23s$ git fetch origin +refs/pull/3/merge:
# ... truncated
$ git checkout -qf FETCH_HEAD
$ source ~/virtualenv/python3.6/bin/activate
$ python --version
# ... truncated
$ pip --version
# ... truncated
$ pip install pip-tools
# ... truncated
$ pytest test_handler.py
============================= test session starts ==============================
platform linux -- Python 3.6.3, pytest-3.4.0, py-1.5.2, pluggy-0.6.0
rootdir: /home/travis/build/joshuaballoch/testing-lambda-py, inifile:
collected 2 items                                                              
test_handler.py ..                                                       [100%]
=========================== 2 passed in 1.59 seconds ===========================
The command "pytest test_handler.py" exited with 0.
Done. Your build exited with 0.
```

This truncated view shows us Travis isn't doing anything magical - it's basically just running a machine, executing the commands we've specified in our configuration file. It installs the dependencies we've specified, checks out our project's code, and then runs our test. It uses the exit code to know whether the build passed or failed.

With this in place, readers that just want continuous integration have all they need. As an extra, if you want to add a "build" sticker to your Readme (or project website), it's easy to add in. It will display the result of the last `master` build directly on your repo. As an example, for my repo I just add:

```
[![Build Status](https://travis-ci.org/joshuaballoch/testing-lambda-py.svg?branch=master)](https://travis-ci.org/joshuaballoch/testing-lambda-py)
```

# A First Automated Deploy

As a quick note, continuous deployment isn't meant for all projects. If you're working in a multi staged setup, then having automatic deploys to pre-production environments can be convenient, simple, and without many risks. On the flipside, automating deploys directly to production is something I rarely see, as this usually involves more risks. Basically, if you're considering CD for a given project, consider if there are questions, steps, or checks you do manually before deploys and whether this can be encapsulated back into the configuration of your automated deploy.

In this project, we'll automate deploys to staging on successful `master` builds. Our Lambda function is pretty simple, and I'd say the tests we wrote give us high confidence that a passing build means a working function.

The simplest way to test out deployment functionality on our app is to add the `serverless deploy` command to our config's `script` section. We'll also need to instruct Travis to install serverless. This won't suffice for our full setup - we only want deploys on the `master` branch - but it will let us try things out. 

```
# ...
install:
  # ...
  - pip install -r ~/requirements-dev.txt
  - npm install -g serverless
script: 
  - pytest test_handler.py
  - serverless deploy --stage staging
```

A quick push to Github will lead to a new build, but Travis doesn't have credentials to push to my AWS account, so the build fails. Here's part of the log [from that failed build](https://travis-ci.org/joshuaballoch/testing-lambda-py/builds/349234015).

```
# ...
$ serverless deploy --stage staging
Serverless: Packaging service...
Serverless: Excluding development dependencies...
 
  Serverless Error ---------------------------------------
 
  ServerlessError: AWS provider credentials not found. Learn how to set up AWS provider credentials in our docs here: http://bit.ly/aws-creds-setup.
 
  Get Support --------------------------------------------
     Docs:          docs.serverless.com
     Bugs:          github.com/serverless/serverless/issues
     Forums:        forum.serverless.com
     Chat:          gitter.im/serverless/serverless
 
  Your Environment Information -----------------------------
     OS:                     linux
     Node Version:           8.9.1
     Serverless Version:     1.26.1
```

We just need to add AWS credentials to our Travis project. This is found in the Settings page of a project ([`https://travis-ci.org/<username>/<repo>/settings`](https://travis-ci.org/<username>/<repo>/settings)), under "Environment Variables". Add your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. Then we can use the "Restart Build" button on our failed build page to trigger a new build.

Now our [build passed](https://travis-ci.org/joshuaballoch/testing-lambda-py/builds/349234015) and our function was deployed to AWS. The serverless output in the Travis logs is the same as when we deploy locally:

```
$ serverless deploy --stage staging
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (30.23 KB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
.........
Serverless: Stack update finished...
Service Information
service: hello-world
stage: staging
region: us-east-1
stack: hello-world-staging
api keys:
  None
endpoints:
  None
functions:
  hello_world: hello-world-staging-hello_world
```

Now that we've tested this out, let's make deployments only occur on master. To do this, move the deploy script into the `deploy` section of config:

```
language: python
python:
  - "3.6"
install:
  - pip install pip-tools
  - pip-compile --output-file ~/requirements-dev.txt requirements/requirements-dev.in
  - pip install -r ~/requirements-dev.txt
  - npm install -g serverless
script:
  - pytest test_handler.py
deploy:
  provider: script
  script: serverless deploy --stage staging
  on:
    branch: master
```

The [next build](https://travis-ci.org/joshuaballoch/testing-lambda-py/builds/349245757) of our branch now no longer attempts to deploy to AWS. Instead it lets us know it's skipping the deploy:

```
# ...
The command "pytest test_handler.py" exited with 0.
Skipping a deployment with the script provider because the current build is a pull request.
Done. Your build exited with 0.
```

# Custom Python Packages

This simple setup now has all the hallmarks of continuous integration & deployment, but minor modifications are required for users with Lambda functions using custom python packages. Deploying Lambda Functions with custom packages is covered in the [second post](/custom-packages-lambda) of [this series](/production-ready-aws-lambda). We'll need to make some changes to our Travis config to achieve the same here.

As in [that previous post](/custom-packages-lambda), let's presume we want to add in `numpy` to our function. We'll make the same modifications into this function as we covered in that previous post (please go there for a full explanation of these additions, and other changes needed to deploy from your local environment). 

Add an `import numpy` to the top of our `handler.py` file, and then add `numpy` inside of `requirements/requirements.in`. Add the following `package.json` file:

```
{
  "name": "test-lamda-fns",
  "description": "",
  "version": "0.1.0",
  "dependencies": {},
  "devDependencies": {
    "serverless-python-requirements": "^3.0.10"
  }
}
```

Then make these additions to the `serverless.yml` config file:

```
plugins:
  - serverless-python-requirements

custom:
  pythonRequirements:
    dockerizePip: non-linux
```

Now we would normally be just around the corner from deploying from our local environment - but today we're looking to deploy from Travis. To do so, we'll need to add two lines to the `install` section of `.travis.yml`. One is to output a `requirements.txt` file for the deploy. The other is to install the `serverless-python-requirements` NPM package:

```
install:
  # ...
  - pip-compile --output-file requirements.txt requirements/requirements-dev.in
  - npm install
```

Let's temporarily comment out the `deploy` section, and just add the `serverless deploy` command to the `script` section. This will let us test deploys on a non-master branch:

```
script:
  - pytest test_handler.py
  - serverless deploy --stage staging
# deploy:
#   provider: script
#   script: serverless deploy --stage staging
#   on:
#     branch: master
```

With these changes in place, Travis should be able to deploy our function, complete with our custom python requirements. A push to Github will trigger a new build, [which should pass](https://travis-ci.org/joshuaballoch/testing-lambda-py/builds/349261276). We can see the deployment is now much larger than before:

```
# ...
$ serverless deploy --stage staging
Serverless: Installing required Python packages with python3.6...
Serverless: Linking required Python packages...
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Unlinking required Python packages...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (30.86 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
.........
Serverless: Stack update finished...
Service Information
service: hello-world
stage: staging
region: us-east-1
stack: hello-world-staging
api keys:
  None
endpoints:
  None
functions:
  hello_world: hello-world-staging-hello_world
Serverless: Removing old service versions...
```

With that working, we can revert the serverless config to have deploys on master only:

```
script:
  - pytest test_handler.py
deploy:
  provider: script
  script: serverless deploy --stage staging
  on:
    branch: master
```

And that's it - turns out, very few changes are needed to have our Travis deployments support custom python packages.

# Conclusion

I hope you've liked this [series on building a production-ready setup in AWS Lambda](/production-ready-aws-lambda). We've built a local development environment, learned how to get custom packages installed and deployed, added in tests & enabled us to TDD our functions, and now have integrated CI/CD as well. These components sum together to a respectably production-ready setup for developing Lambda functions.

If you have any ideas on additions or alterations to this series, let me know via the comments below or by email!

For those interested, the final code from this post is available here:

- [Travis CI/CD setup without custom python packages](https://github.com/joshuaballoch/testing-lambda-py/tree/ci-cd)
- [Travis CI/CD setup with custom python packages](https://github.com/joshuaballoch/testing-lambda-py/tree/ci-cd-custom-py)
