---
layout: post
title: Sending Notifications from your app using Telegram
tags: testing
date: 2022-06-13
---

In many different domains, it is common to have an application that does some kind of monitoring. In general, these applications periodically check the status of something, and in case there is anything wrong, they should send a warning to registered users.

The choice of an appropriate medium to send that warning can be a little bit of a challennge. Email is a common choice, but nowadays people receive A LOT of email, and it is very likely that your user will not have notifications enabled on his phone for incoming emails. Whatsapp is very popular, but it does not have a free API for your application to interact with.

### Enter Telegram

Despite not being as popular as Whatsapp, Telegram offers some interesting features. One of them is an open API that anyone can use for free. And how can you use it to send notifications? This is what this post is all about.

Firstly, it is important to notice that you cannot use the API to send messages on behalf of a regular telegram user. That is why we will be using a bot as the sender. In a nutshell, the required steps are:

1. Register your bot

2. Provide some means for the users to register themselves.

3. Use the bot to send notifications

4. Perfumarias (menu inicial com instruções)


### Step 1 - Creating Your Bot

Let's create the bot to use in this example. You will need a personal telegram account, so the first step is to create one, if you do not already have.

Then, from the telegram app, initiate a conversation with @botfather. Botfather is the bot responsible for creating other bots. You will be prompted to choose a name and username for your bot.

<div class='message'>
The name of your bot is displayed in contact details and elsewhere.

The Username is a short name, to be used in mentions and t.me links. Usernames are 5-32 characters long and are case insensitive, but may only include Latin characters, numbers, and underscores. Your bot's username must end in 'bot', e.g. 'tetris_bot' or 'TetrisBot'.
</div>

Once your bot has been created, you will receive a token.

<div class='message'>
The token is a string along the lines of 110201543:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw that is required to authorize the bot and send requests to the Bot API. Keep your token secure and store it safely, it can be used by anyone to control your bot.
</div>

At this point, your bot is already available for conversations to anyone in the world. However, it will not do anything useful unless you program it. Let the fun begin...

### Step 2 - Registering Users

Most telegram bots are open for public use. Anyone can interact with them. You just start a conversation, and it will show you a menu with the available commands.

In this post, we are assuming that the notifications from your app are *not public*, so some kind of authorization should be necessary before the user can receive messages. It is up to you to define how to handle it, but you have to follow one basic restriction: bots can't initiate conversations with users. A user must either add them to a group or send them a message.

Our sample application will work like this:

- The user starts a conversation with the requesting registration
- The bot sends a confirmation link, that points to the web app
- The user opens the link (which requires him to be signed in) to confirm the registration
- His Telegram id is saved in a list of recipients

Finally, let's see some example code. Considering the workflow above, the first thing to do is handle the incoming registration requests. Telegram gives you 2 ways of doing it: polling and webhooks.

Polling is done by calling the [getUpdates](https://core.telegram.org/bots/api#getupdates) method, which will return a list of incoming messages. This is more convenient to use in development, since it does not require a public IP address, but for it to work in production you would need to set up a background process that does the polling. 

If you choose to use a [webhook](https://core.telegram.org/bots/api#setwebhook), you set up a path in your application to receive the messages sent to the bot. Whenever there is an update for the bot, Telegram will make an HTTPS POST request to the specified URL.

The JSON format of the message is the same in both cases, so it should be easy to swich beteween them as needed. But you cannot have both active simultaneously.

Let's start coding! We will be using Django on this example. Start a new Django app:

{% highlight bash %}
django-admin startproject cool_monitor
{% endhighlight %}

Add an app to your Django project to handle the telegram stuff:

{% highlight bash %}
python manage.py startapp notifications
{% endhighlight %}

Open the file `notifications/views.py` and setup the view that will handle the webhook:

{% highlight python %}

{% endhighlight %}

# Step 3 - Sending Notifications

# Step 4 - "Perfumarias"

start menu