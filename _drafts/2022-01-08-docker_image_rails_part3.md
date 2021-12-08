---
layout: post
title: Building a Docker Image for your Rails Application - Part 3
date: 2022-01-08
tags: Docker Rails CI
categories: Docker
---

<!-- intro -->

This is part 2 of a 3-part series on building a Docker image for your Ruby on Rails application. If you missed part 1, check it out [here]({% post_url 2021-11-03-docker_image_rails_part1 %}).

### Install The Application

So far, we have covered environment setup. Now we will start installing the application itself. 

2. Install the application
- Install *ruby* dependencies
- Install *javascript* dependencies
- Copy application files
- Precompile the assets

First, let's install the ruby dependencies:

{% highlight docker %}
COPY Gemfile .
COPY Gemfile.lock .
RUN bundle
{% endhighlight %}
You might be thinking: why not just copy all the application files and then run bundle install?
Well, the docker build process executes a series of commands based on a context. The context, in this case, are the files that you include with the COPY command. If the context does not change between builds, docker will use a cached result instead of executing the command again. If we had copied the entire application in this step, whenever you changed any file, the build process would reinstall all the gems. Copying only the bundle-related files has the effect of invalidating the docker build cache only when the bundle changes.

Continuing with the setup, install javascript dependencies:

{% highlight docker %}
COPY package.json .
COPY yarn.lock .
RUN yarn
{% endhighlight %}

Then, we copy the application files:

{% highlight docker %}
RUN mkdir -p /app
WORKDIR /app
COPY . /app
{% endhighlight %}

There is one important thing to note here, tough. When docker runs the command `COPY . /app`, it will copy *all* files it finds in the application directory. If you are not careful, this might include log files (in case you are logging to the filesystem when developing), database files (if stored under /app), and other unnecessary stuff. Including unwanted files in your image can cause you two problems: it will increase the image size, and every time one of these file changes, it will trigger an unnecessary rebuild of the image

To solve this, you should have a `.dockerignore` file that tells docker which files you *do not* want in the resulting image. You can read about it [here](https://docs.docker.com/engine/reference/builder/#dockerignore-file), if you are not familiar, or grab the [example](https://github.com/CoolRequest/rails_docker_demo/blob/master/.dockerignore) from our rails_docker_demo repository.


Next step, the application's assets. Since we will be hosting the assets on the same container, we have to get them into the image:

{% highlight docker %}
RUN bundle exec rake assets:precompile DB_ADAPTER=nulldb NODE_ENV=development RAILS_ENV=staging SECRET_KEY_BASE=123
{% endhighlight %}

The `DB_ADAPTER=nulldb` variable is worth mentioning. When rails runs this rake task, it will boot the application in order to run the compiling. Depending on your environment, you might not have the environment varibles with database connection parameters at this moment, which would result in an error. So we use the [nulldb](https://github.com/nulldb/nulldb) adapter, database backend that translates database interactions into no-ops.

Finally, the EXPOSE command informs the TCP port the container will be responding on, and CMD provides a default command for running the server:

{% highlight docker %}
EXPOSE 3000
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
{% endhighlight %}

### That's It
These are the steps necessary to make a Docker image from your Rails app. You can access the full Dockerfile in [this link](https://github.com/CoolRequest/rails_docker_demo/blob/master/Dockerfile).
This sample project also contains a `docker-compose.yml` file that starts the application and a containerized *postgresql* database that you can use to try it out.

---

<div class="message">
If you want to run your application on Docker but lack the necessary time or experience, feel free to contact us and we will be happy to give you a hand.
</div>
