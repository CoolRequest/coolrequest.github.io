---
layout: post
title: Dockerizing Rails Applications
---

<!--
Pendencias da Parte 1:
- colocar a aplicação demo num repositório coolrequest (rails_docker_demo) e acertar os links do Dockerfile e .dockerignore
- explicar por que nulldb no assets precompile
- falar em como setar o environment
- falar em RAILS_LOG_TO_STDOUT
- criar um texto de fechamento para o final do post
- ressaltar por que as coisas que não mudam tem que vir antes
- falar em dividir o dockerfile em dois (o "base com banco" e o da aplicação)
- colocar também um docker-compose de exemplo no repositório
- depois que terminar, seguir os passos do inicio ao fim e ver se funciona (rodar o build e subir a aplicacao)

** DONE
- instalação do postgresql não está funcionando
- faltou falar do .dockerignore
- rever comando para instalação do yarn
- ajustar comando do bundle

Backlog Para a parte 2: --------------------------------------------------------------
- rodar yarn antes de copiar a app
- rodar bundle antes de copiar a app
- explicar o rolo do bundle “deployment”
- mudando para multi stage build (talvez não faça sentido agora com as gems novas)
- instalando algumas gems manualmente antes do bundle
-->

<!--intro-->
Wether you are running apps on your own infrastructure or deploying to the cloud, there are many [reasons](https://www.docker.com/why-docker) to containerize your Rails application. However, the rails new template doesn’t help you there, as the default generated code needs to be adapted to run in Docker. You will need to do some modifications to the app and add a carefully designed Dockerfile, which is important for speeding up build times and reducing the image size. That's what this post is about.

This text is divided in two parts. Part one will cover the basics, and show how to write a simple Dockerfile that should work for most apps. Part two will explain some tweaks that you can use to make your build faster and the resulting image lightweight. A basic understanding of Docker concepts and Dockerfile syntax is required for understanding the content.

One very important decision, that will impact the way you build your image is how you want to handle the assets. Particularlly, you should answer these two questions:
1. When will the assets precompilation happen? Will it be done in development time, and pushed to the source code repository? Or will it take place later, during the deploy process?
2. How will these assets be served? By the same application server? Other server on the same infrastructures? Or will you use an external CDN?

There are no correct answers here, as this depends on many factors that are application or environment-specific. In our use case, which is described here, we assume a simple scenario where:
- We don't want developers to care about building assets for production. The CI script should handle it.
- In a small scale environment, the assets can be served by the same container that runs the application.

<!-- configs -->
Now let's get started. The first thing to be aware of when containerizing your application is that you should not have any configuration data in your docker image. This means that you will have to look at your application source code searching for files that hold database connection settings, URLs for external services, and any other settings that might change depending on the environment. Typical files to look are `config/database.yml`, `config/storage.yml`, and `config/initializers/*`. You should replace hard-coded values with environment variables. Here is what a typical `database.yml` would look like, using postgresql as an example:

```
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 20) %>
  database: <%= ENV["DB_DATABASE"] %>
  username: <%= ENV["DB_USERNAME"] %>
  password: <%= ENV["DB_PASSWORD"] %>
  host: <%= ENV["DB_HOST"] %>
  port: <%= ENV["DB_PORT"] || 5432 %>

development:
  <<: *default

test:
  <<: *default

staging:
  <<: *default

production:
  <<: *default
```

<!--dockerfile v1-->
Now that the configuration data is taken care of, let’s look at the steps necessary for building the image that will run the application. These are the commands that will go in the application's Dockerfile. Starting from a very skinny linux distribution, we need to install all the things necessary to precompile the assets and run the app. In summary, what needs to be done is:
1. Setup the environment
- Choose a base image to start from
- Install basic build tools
- Install *nodejs*, *yarn*, *bundler*
- Install database client
2. Install the application
- Copy application files
- Install *ruby* dependencies
- Install *javascript* dependencies
- Precompile the assets

Let's go over these, step by, step. We start from the official ruby Docker image:
```
FROM ruby:2.7.4-slim
```

Then, add basic build tools which will be required in subsequent steps:
```
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
```

We will be needing *nodejs* to handle assets precompiling, *yarn* to install javascript dependencies, and *bundler* for the ruby gems:
```
RUN curl -sL https://deb.nodesource.com/setup_lts.x | bash - && \
    apt-get update && \
    apt-get install -y --no-install-recommends nodejs && \
    rm -rf /var/lib/apt/lists/*

RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - && \
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends yarn && \
    rm -rf /var/lib/apt/lists/*

RUN gem install bundler
```

Next step, database client. You will need to install the client library / headers. This part is highly dependable on the database you are using. Here is what you would need for *postgresql*:
```
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*
```

So far, we have covered environment setup. Now we will start installing the application itself. First, we coppy the application files:
```
RUN mkdir -p /app
WORKDIR /app
COPY . /app
```
There is one important thing to note here, tough. When docker runs the command `COPY . /app`, it will copy *all* files it finds in the application directory. If you are not careful, this might include log files (in case you are logging to the filesystem when developing), database files (if stored under /app), and other unnecessary stuff. Including unwanted files in your image can cause you two problems:
- It will increase the image size
- Every time one of these file changes, it will trigger an unnecessary rebuild of the image

To solve this, you should have a `.dockerignore` file that tells docker which files you *do not* want in the resulting image. You can read about it [here](https://docs.docker.com/engine/reference/builder/#dockerignore-file), if you are not familiar, or grab the [example](url_for_dockerignore_here) from our rails_docker_demo repository.

Then, install ruby dependencies:
```
RUN bundle install
```

.. and javascript dependencies:
```
RUN yarn
```

The assets are going to be precompiled during the image build process:
```
RUN bundle exec rake assets:precompile DB_ADAPTER=nulldb NODE_ENV=development RAILS_ENV=staging SECRET_KEY_BASE=123
```

<!-- fechamento -->
That's it. Now you can run your Rails application inside a Docker container, be it for development or production. You can access the full Dockerfile in [this link](https://github.com/coolrequest/docker_demo/Dockerfile).
Stay tuned for part 2 of this series, we will show some optimizations that can be done to this basic Dockerfile.
