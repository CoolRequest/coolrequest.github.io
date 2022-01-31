---
layout: post
title: Building a Docker Image for your Rails Application - Part 3
date: 2022-01-31
tags: Docker Rails CI
categories: Docker
---

This is the last part of the series on building a Docker image for your Ruby on Rails application.

- [Part 1]({% post_url 2021-11-03-docker_image_rails_part1 %}) is about making your application suitable to run inside a docker container.
- [Part 2](2021-12-13-docker_image_rails_part2) covers building a base image that contains the prerequisites needed for a typical Ruby on Rails app.
- Part 3 (this post) - is about installing the application in the base image and running it as a container

### Intro

In the previous posts, we have set up a base image with the environment to run a Rails app. Now we will start installing the application itself. The following steps will be necessary:
- Install Ruby gems
- Install *javascript* dependencies
- Copy application files
- Precompile the assets
- Run the app

### Install the Ruby gems:

The following lines in the Gemfile will do the trick:

{% highlight docker %}
COPY Gemfile .
COPY Gemfile.lock .
RUN bundle
{% endhighlight %}

You might be thinking: why not just copy all the application files and then run bundle install?
Well, the docker build process executes a series of commands based on a context. The context, in this case, are the files that you include with the COPY command. If the context does not change between builds, docker will use a cached result instead of executing the command again. If we had copied the entire application in this step, whenever you changed any file, the build process would reinstall all the gems. Copying only the bundle-related files has the effect of invalidating the docker build cache only when the bundle changes.

### Install *javascript* dependencies

For the same reason above, we just copy the *yarn*-related files before installing *javascript* dependencies:

{% highlight docker %}
COPY package.json .
COPY yarn.lock .
RUN yarn
{% endhighlight %}

### Copy the application files:

The following commands will create a folder called *app* inside the image, and copy your application:

{% highlight docker %}
RUN mkdir -p /app
WORKDIR /app
COPY . /app
{% endhighlight %}

There is one important thing to note here, tough. When docker runs the command `COPY . /app`, it will copy *all* files it finds in the application directory. If you are not careful, this might include log files (in case you are logging to the filesystem when developing), database files (if stored under /app), and other unnecessary stuff. Including unwanted files in your image can cause you two problems: it will increase the image size, and every time one of these file changes, it will trigger an unnecessary rebuild of the image

To solve this, you must have a `.dockerignore` file that tells docker which files you *do not* want in the resulting image. You can read about it [here](https://docs.docker.com/engine/reference/builder/#dockerignore-file), if you are not familiar, or grab the [example](https://github.com/CoolRequest/rails_docker_demo/blob/master/.dockerignore) from our [rails_docker_demo](https://github.com/CoolRequest/rails_docker_demo) repository.


### Precompile the assets

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

### Build the image

Our Dockerfile is now ready. To build the image, execute the *docker build* command:

{% highlight bash %}
docker build -t coolrequest/rails_docker_demo .
{% endhighlight %}

This will execute the commands in the Dockerfile, using the current directory as context, save the image in your local Docker repository and tag it with the name *coolrequest/rails_docker_demo*

### Run the app

Our application image is now ready. All you need to do to run it is provide the appropriate values for the environment variables. One simple way to do that is using a *docker-compose* file.

Suppose you don't have *postgres* installed and want to run it using Docker as well. Create a *docker-compose.yml* file like this one:

{% highlight yaml %}
version: '3'
services:
  
  rails_docker_demo:
    image: coolrequest/rails_docker_demo
    ports:
      - 3000:3000
    depends_on:
      - db
    env_file:
      - .env
  
  db:
    image: postgres
    restart: always
    environment:
      POSTGRES_USER: rails_docker_demo
      POSTGRES_PASSWORD: my_pg_pass123
    volumes:
      - ./db/pgdata:/var/lib/postgresql/data
    ports:
      - 5432:5432
{% endhighlight %}

... and run it using this command line:
{% highlight bash %}
docker compose up
{% endhighlight %}


### That's It
These are the steps necessary to make a Docker image from your Rails app.
Using a similar approach, you could run it on a docker swarm or kubernetes cluster.
Thanks for reading, and feel free to leave your thoughts on the comments section below.


---

<div class="message">
Check out our repository with the full example here:<br/>
<b>
<a href="https://github.com/CoolRequest/rails_docker_demo/blob/master/Dockerfile">https://github.com/CoolRequest/rails_docker_demo/blob/master/Dockerfile</a>
</b>
</div>
