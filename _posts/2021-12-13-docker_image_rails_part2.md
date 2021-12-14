---
layout: post
title: Building a Docker Image for your Rails Application - Part 2
date: 2021-12-13
tags: Docker Rails CI
---

<!-- intro -->

This is part 2 of a 3-part series on building a Docker image for your Ruby on Rails application.

- [Part 1]({% post_url 2021-11-03-docker_image_rails_part1 %}) is about making your application suitable to run inside a docker container.
- Part 2 (this post) covers building a base image that contains the prerequisites needed for a typical Ruby on Rails app.
- Part 3 (coming soon!) - describes  how to add the application files, create and run your container.


### Preparing the Environment
The image you will be building should contain all the dependencies needed to install the gems, instal the javascript packages, precompile the assets and run the app. In summary, these are the steps to follow:
1. Choose a base image to start from
2. Install *nodejs* runtime
3. Install *yarn*
4. Install the database client

Let's go over these steps, one at a time.

<!-- 
To be reviewed from here on.
As it was broken in parts, there is room to write more about these steps.
Add some comments / explanations on each one.
-->


### The Base Image
We start from the official ruby Docker image:
{% highlight docker %}
FROM ruby:2.7.4
{% endhighlight %}

This tag, by design, contains the most common Debian packages, which you will need later to compile the gems that have native extensions.

### Install *nodejs* runtime

We will be needing *nodejs* to handle assets precompiling. Here is how to install it from *nodesource*:
{% highlight docker %}
RUN curl -sL https://deb.nodesource.com/setup_lts.x | bash - && \
    apt-get update && \
    apt-get install -y --no-install-recommends nodejs && \
    rm -rf /var/lib/apt/lists/*
{% endhighlight %}

Note that this is a single `RUN` command that executes multiple shell operations: installs the repository source, downloads package lists, installs *node* and removes the package lists. There is a reason to do it this way.

A Docker image is built up from a series of [layers](https://docs.docker.com/storage/storagedriver/#images-and-layers). Each docker RUN command that modifies the image contents creates an additional layer on the filesystem. If we had done the above operation on 4 steps, it would result in 4 layers that have take storage space on your system. Doing the 4 operaions in one step results in only one layer which has the files relevant to run *node*.

### Install *yarn*

`Yarn` will be necessary to install javascript dependencies. 
The installation, using `npm`, is pretty straightforward:

{% highlight docker %}
RUN npm install --global yarn
{% endhighlight %}

### Database Client

Next step, database client. You will need to install the client library and header files. This will, of course, be completely different depending on the database you choose. Here is what you would need for *postgresql*:

{% highlight docker %}
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*
{% endhighlight %}

### That's It

We've seen how to prepare a Dockerfile with the typical prerequisites for a Rails application. In the next post, we put things together by installing our container-ready application on the base image. Stay tuned!