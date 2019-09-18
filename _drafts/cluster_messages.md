---
layout: post
title: Setting up convenient notifications from the cluster
categories: [tools]
tags: [cluster, linux]
menuid: "Tools"
---

Recently I started to work with the computer cluster again, and bumped into one thing that I always found particularly annoying -- getting notifications from my program when it runs on a remote machine. For some reasons, email notifications never worked for me. And now I think that I have found a super simple and super efficient workaround!

## The problem
From time to time, I run my script, say, written in Python, on a remote machine (via these standard `qsub` and other commands). The problem is, quite often it takes pretty long time to run -- and I want to know what's going on. At the very least, I want to know when my computation proceeds to some next logical step (say, runs the next script) and ideally, I want to be able to recieve some notifications from within the code -- if something really important happens. Ideally, to my mobile phone. For example, last time my code was looking for counter-examples to some hypotheses, and it was important to notify me about the cases found -- so I could start looking into these in parallel.

**A straightforward approach:** log in via ssh and check what's going on. It is relatively convenient, but still takes too much time: a word in the command line, paste the password, press a button on the mobile phone for the two-factor authentication, then at least one command to see if there is any output. Plus it takes some time to connect. **A standard solution:** there is [one](https://www.palmetto.clemson.edu/palmetto/userguide_basic_usage.html) (search for the word "mail" on that page) -- which, for some reason, never worked for me. I think I tried to use it on two different clusters, as a user -- never got a single email. This way feels pretty limited anyways, since I could theoretically get an email only when the job starts, ends or aborts.

I had several restrictions:
- little to no 3-rd party software (I'm not sure to what extent I can install stuff on the cluster);
- no 3-rd party libraries for the programming language I use (what if, e.g., there's no package for Julia);
- NO COMPLICATED SET UP -- I just don't have time for this. Like, no complicated APIs.

I was thinking about the matrix.org client-server [API](https://matrix.org/docs/guides/client-server-api) to employ, since it is free and open source, and all that glory -- but it looked like too much effort for just a nice-to-have thing. But then, out of sudden, I have stumbled upon a solution from a completely unexpected side: [Telegram](https://telegram.org/faq#q-what-is-telegram-what-do-i-do-here). Yes, it's a messenger that claims to be ``super-fast, simple and free''. I use it a lot, and find it, indeed, pretty convenient -- but hey, all the messengers are not that different. Or are they?

## Step zero: it's alive!
There is something that is called [Bots](https://telegram.org/faq#bots). Non-playing characters. Computer programs you can chat with. And the secret is the entry barrier is extremely low. Apparently, you can set up one in 5 minutes (literally) -- and use it with virtually no additional code. Let me share my experience (assuming you already have Telegram [app](https://telegram.org/apps) installed on your phone, tablet, desktop, albeit a Mac, Win, Linux -- or wherever you prefer to have it).

First, let us create a bot. Go to the app, find a "user" called [BotFather](https://telegram.me/botfather) -- this is actually a bot for creating bots (a recursive bot, yes. Note that if you have an app installed -- you can just click the previous link, it will open it correctly). Tell it, just in a chat window:
```
you: /start
...
you: <bot-name>
...
you: <bot-username>
```

Like, that's it. You have a personal bot. Write down / save somewhere the "token" BotFather will give you and the bot username -- will be needed to access the bot. *bot-name* here is a human-readable name you will see in the messenger when it will be messaging you. *bot-username* is a unique ID of your bot (should end with 'bot'). I have created a bot called `Wintermute`, but since the username `wintermute_bot` was already taken, I used `ab_wintermute_bot`. So my (literally) one-minute conversation with BotFather looked as follows:

<div style="width:50%; margin: 0px auto;">
![Creating a bot in Telegram](/assets/tgpost/botfather.png)
</div>

Now, we need to do the only piece of magic in the whole process. Find your bot by the username you have just given it (for example, I have typed `@ab_wintermute_bot` into the "Search" field in the Telegram app). Open a chat and tell it `/start`. I also needed to tell something else -- like, `test`. The thing will listen but will not respond (obviously, since we haven't implemented anyone who could respond). We need this to obtain a so called "chat ID": now, open up your browser and open the URL: https://api.telegram.org/botAAA:BBB/getUpdates, replacing *AAA:BBB* with your "token ID" given by the `BotFather`. Look for the `chat id` and write it down somewhere:

<div style="width:50%; margin: 0px auto;">
![Chat ID](/assets/tgpost/chat_id.png)
</div>

Essentially, this is everything we need to start chatting with our remote scripts.

## Step one: talk to me
First, let us create a short script to send messages to ourselves. Open up your favorite text editor and copy the following, replacing \<your-token\> and \<your-chat-id\> with the values you obtained in the previous section:

```bash
#!/bin/bash

TOKEN=<your-token>
CHAT_ID=<your-chat-ID>
read MESSAGE
URL="https://api.telegram.org/bot$TOKEN/sendMessage"

curl -s -X POST $URL -d chat_id=$CHAT_ID -d text="$MESSAGE"
```

Save it as, for example, `tgsend.sh`. Note that you will need to make it executable on a remote machine. That is, after you copy it to your remote folder, do the standard:

```bash
$ chmod u+x ./tgsend.sh
```

Done! You are ready to send a messages to your mobile phone or tablet! You can try it on your login shell first:

```bash
$ echo Hello from the login shell | ./tgsend.sh
```

You should have got a message from yourself (exciting, isn't it?) So, since it works, you can use it in your `PBS` files that you feed to `qsub` to start a remote job. For example, in your `myjob.pbs` you could write something like:

```bash
#PBS -N AwesomeJobName
#PBS -l select=1:ncpus=1:mem=2gb,walltime=06:00:00

module add anaconda3/5.1.0

cd /home/$USER
echo "START: my awesome job..." | ./tgsend.sh
python ./my_awesome_script.py > awesome.log
echo "Done with the calculation! Please log in and check" | ./tgsend.sh
```

At least now we have notifications about start and end of the job, delivered to your mobile phone. Don't have those if the job was killed (memory limit exceeded, walltime limit exceeded, etc.). For me it was a minor issue -- but I'd appreciate feedback if you know an easy fix for this. As an ugly workaround, if it is critically important, you could always ask your script to ping you, say, every half an hour with "I'm alive" kind of message.


## Step two: text me if anything happens
Now, this is already pretty flexible. However, you might want you script to text you in the middle of execution. For example, if my script is looking for certain examples, it would be nice if it texted me "hey, look at the instance No. X!" and continued searching. In Python it [can](https://medium.com/@ManHay_Hong/how-to-create-a-telegram-bot-and-send-messages-with-python-4cf314d9fa3e) be done as follows (again, don't forget to fill in your token and chat id):

```python
# sends a message to my bot
import requests

def tg_sendtext(bot_message):
    bot_token = '<your-token>'
    bot_chatID = '<your-chat-id>'
    send_text = 'https://api.telegram.org/bot' + bot_token + '/sendMessage?chat_id=' + bot_chatID + '&parse_mode=Markdown&text=' + bot_message
    response = requests.get(send_text)
    return response.json()
```

Then, when you need to text yourself, just do the obvious:
```python
tg_sendtext("Counter-example found: instance no. {}".format(instance_no))
```
I was working with Python, but if you use Julia it also shouldn't be a problem:

```julia
using HTTP

function tg_sendtext(msg)
    bot_token = "<your-token>"
    bot_chatID = "<your-chat-id>"
    send_text = "https://api.telegram.org/bot$(bot_token)/sendMessage?chat_id=$(bot_chatID)&parse_mode=Markdown&text=$(msg)"
    return HTTP.request("GET",send_text)
end
```

would also give you a function to text a string to yourself from a remote machine (you'll need a package `HTTP`, obviously. Tested with Julia 1.0.3 on a local machine -- but since python did not require anything on top of `module add anaconda...` and `import requests`, I would assume it should be pretty standard).

And so, with around 20 additional lines of code I have all this glory:

<div style="margin: 0px auto;">
![chat screen](/assets/tgpost/chat_screen.png)
</div>
