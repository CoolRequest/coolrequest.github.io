---
layout: post
title: Building a Docker Image for your Rails Application - Part 2
date: 2021-12-08
tags: Docker Rails CI
categories: Docker
---

<!-- intro -->

This is part 2 of a 3-part series on building a Docker image for your Ruby on Rails application.

- [Part 1]({% post_url 2021-11-03-docker_image_rails_part1 %}) is about making your application suitable to run inside a docker container.
- Part 2 (this post) covers building a base image that contains the prerequisites needed for a typical Ruby on Rails app.
- Part 3 (coming soon!) - describes  how to add the application files, create and run your container.

<!-- 
To be reviewed from here on.
As it was broken in parts, there is room to write more about these steps.
Add some comments / explanations on each one.
-->

### Preparing the Environment
Starting from a very skinny linux distribution, we need to install all the things necessary to precompile the assets and run the app. In summary, what needs to be done is:
1. Choose a base image to start from
2. Install basic build tools
3. Install *nodejs* runtime
4. Install package managers (*yarn*, *bundler*)
5. Install the database client

Let's go over these, step by, step.

### The Base Image
 We start from the official ruby Docker image:
{% highlight docker %}
FROM ruby:2.7.4-slim
{% endhighlight %}

### Basic Build Tools

Then, add basic build tools which will be required in subsequent steps:
{% highlight docker %}
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
              ca-certificates \
              curl \
              git \
              openssh-client \
              build-essential \
              libc6 \
              gnupg \
              shared-mime-info && \
    rm -rf /var/lib/apt/lists/*
{% endhighlight %}

### Install *nodejs* runtime

We will be needing *nodejs* to handle assets precompiling, *yarn* to install javascript dependencies, and *bundler* for the ruby gems:
{% highlight docker %}
RUN curl -sL https://deb.nodesource.com/setup_lts.x | bash - && \
    apt-get update && \
    apt-get install -y --no-install-recommends nodejs && \
    rm -rf /var/lib/apt/lists/*

### Install package managers (*yarn*, *bundler*)

RUN npm install --global yarn

RUN gem install bundler
{% endhighlight %}

### Database Client

Next step, database client. You will need to install the client library / headers. This part is highly dependable on the database you are using. Here is what you would need for *postgresql*:

{% highlight docker %}
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*
{% endhighlight %}

### That's It

We've seen how to prepare a Dockerfile with the typical prerequisites for a Rails application. In the next part, we put things together by installing our container-ready application on the base image. Stay tuned!