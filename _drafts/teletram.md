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

1. Create your bot

2. Choose a registration workflow for new users

3. Prepare your Rails Application

4. Set up webhook

5. Implement registration logic

6. Use the bot to send notifications

7. Perfumarias (menu inicial com instruções)


### Step 1 - Creating Your Bot

Let's create the bot to use in this example. You will need a personal telegram account, so the first step is to create one, if you do not already have.

Then, from the telegram app, initiate a conversation with @botfather. Botfather is the bot responsible for creating other bots. You will be prompted to choose a name and username for your bot.

<div class='message'>
The <em>name</em> of your bot is displayed in contact details and elsewhere.

The <em>username</em> is a short name, to be used in mentions and t.me links. Usernames are 5-32 characters long and are case insensitive, but may only include Latin characters, numbers, and underscores. Your bot's username must end in 'bot', e.g. 'tetris_bot' or 'TetrisBot'.
</div>

Once your bot has been created, you will receive a token.

<div class='message'>
The token is a string along the lines of 110201543:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw that is required to authorize the bot and send requests to the Bot API. Keep your token secure and store it safely, it can be used by anyone to control your bot.
</div>

At this point, your bot is already available for conversations to anyone in the world. You should be able to  open the telegram app and send a message to it. However, you won't receive any reply. Not surprising, since we have not programmed it yet. Let the fun begin...

### Step 2 - Define a registration workflow

Most telegram bots are open for public use. Anyone can interact with them. You just start a conversation, and it will show you a menu with the available commands.

In this post, we are assuming that the notifications from your app are *not public*, so some kind of authorization should be necessary before the user can receive messages. It is up to you to define how to handle it, but you have to follow one basic restriction: bots can't initiate conversations with users. A user must either add them to a group or send them a message.

Our sample application will work like this:

- The user starts a conversation with the bot and sends a "/authorize" command requesting registration
- The bot sends a confirmation link, that points to the web app
- The user opens the link (which requires him to be signed in) to confirm the registration
- His Telegram id is saved in a list of recipients

### Step 3 - Preparing the Ground

If you are adding notifications to an existing application, you can skip this step. But, since this tutorial is meant to be complete and reproducible, we will start a new Rails App from strach.

{% highlight bash %}
rails new telegram_demo
{% endhighlight %}

Choose some database to work with (postgresql should be fine).

We will be using [devise](https://github.com/heartcombo/devise) to handle user authentication. This setup is out of the scope of this post, but there are plenty of tutorials on that.

### Step 3 - Set up webhook

Now let's start the telegram integration. Considering the workflow above, the first thing to do is handle the incoming registration requests. Telegram gives you 2 ways of doing it: polling and webhooks.

Polling is done by calling the [getUpdates](https://core.telegram.org/bots/api#getupdates) method, which will return a list of incoming messages. This is more convenient to use in development, since it does not require a public IP address, but for it to work in production you would need to set up a background process that does the polling. 

If you choose to use a [webhook](https://core.telegram.org/bots/api#setwebhook), you set up a path in your application to receive the messages sent to the bot. Whenever there is an update for the bot, Telegram will make an HTTPS POST request to the specified URL.

The JSON format of the message is the same in both cases, so it should be easy to swich beteween them as needed. But you cannot have both active simultaneously.

We'll set up a route and a controller for the telegram webhook. Pretty easy.

{% highlight ruby %}
# config/routes.rb
Rails.application.routes.draw do
  post 'telegram/webhook'
end
{% endhighlight %}

{% highlight bash %}
rails g controller telegram
{% endhighlight %}

{% highlight ruby %}
# app/controllers/telegram_controller.rb
class TelegramController < ActionController::API
  def webhook
    # whenever someone sends a message to the bot, this action gets called
    Rails.logger.info 'message received'
  end
end
{% endhighlight %}

The last thing we need to do before running our first smoke test is to tell Telegram the URL of our webhook. To do that, we use the [setWebhoook](https://core.telegram.org/bots/api#setwebhook) method, which we can call from the command line using cURL:

{% highlight bash %}
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=https://blooming-crag-41150.herokuapp.com/telegram/webhook"

{"ok":true,"result":true,"description":"Webhook was set"}%
{% endhighlight %}

Remember to replace *\<YOUR_BOT_TOKEN\>* in the command with the token you received when creating your bot. Note that the "bot" string that appears before the token value is required, as it is part of the syntax for calling Telegram bot commands.

At this point, you can deploy your app to your favourite hosting provider, and send a message to your bot using the telegram App. Look at the application logs and you should see the webhook getting called with the message data in the request parameters:

{% highlight ruby %}
{
    "update_id"=>543829407,
    "message"=>{
        "message_id"=>18,
        "from"=>{
            "id"=>5366032645,
            "is_bot"=>false,
            "first_name"=>"Mauricio",
            "last_name"=>"Menegaz",
            "language_code"=>"en"
        },
        "chat"=>{
            "id"=>5366032645,
            "first_name"=>"Mauricio",
            "last_name"=>"Menegaz",
            "type"=>"private"
        },
        "date"=>1655722808,
        "text"=>"halo"
    },
    ...
}
{% endhighlight %}

### Step 4. Implement registration logic

The format of the update message we saw in the smoke test is described [here](https://core.telegram.org/bots/api#update) in the official docs. For the time being, we only need to worry about 2 fields:
- message['chat']['id'] - contains a unique number that identifies the conversation. We will associate this chat_id with the user, since it will be necessary to send messages.
- message['text'] - the text of the message or command sent by the user

In the webhook, whenever the bot receives a registration request, what we want to to is:
1. Check if that chat_id is already registered
2. In case it is not, save a registration request and send a link to the user (the request will be pending until the user confirms)



### Step 5 - Sending Notifications

### Step 6 - "Perfumarias"

start menu