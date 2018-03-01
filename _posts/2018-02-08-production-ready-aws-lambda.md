---
layout: post
title: Production-Ready AWS Lambda (Series)
description: "Announcing a new series of posts on getting up and running with AWS Lambda in a production-ready way."
modified: 2018-02-08
tags: [python, lambda, aws-lambda, aws, python3, serverless]
comments: true
subscribe_cta: true
enable_scroll_tracking: true
share: true
---

Serverless functions, like AWS Lambda, provide an easy way to get started building services with just a few lines of code. You can get started directly in the browser - pop in a few lines of code, link it up with other AWS services, and boom! You're done.

If you’re building something production-oriented though, you need more than that. You’ll need to continue to deploy changes to your function. You'll want to be able to push it to a pre-production environment before a prod deploy, and it's a good idea to introduce automated testing. Plus, nobody wants to be editing production code directly in the browser.

In this series, I'll explore building a production-ready workflow AWS Lambda functions. [The first post](/up-and-running-lambda) will get users set up with a basic development environment, complete with one-line deploys using [Serverless](https://serverless.com/). 

In my [next post](/custom-packages-lambda), I'll show users how to improve that setup to support deploying custom python packages with your Lambda functions. In my [third post](/testing-lambda-functions) of this series, I'll walk through how to write automated tests for Lambda. Finally, I'll look into using CI tools to automate test builds and deployments.

As most of my experience with Lambda has been from more of a data science / data engineering angle, we'll explore using Python as the Lambda function's runtime language.

If anyone has feedback on other things to work into this series, let me know via the comments section below!
