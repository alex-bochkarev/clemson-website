---
layout: post
title: Making a remote script to text me - a setup for convenient notifications from the cluster
categories: [tools]
tags: [cluster, linux]
menuid: "Tools"
---

Recently I started to work with the computing cluster again and to run scripts on remote nodes. So I got back to that problem of getting the information back from my program. I haven't found a convenient way to use email for that, but now I think I have got a super simple and super efficient workaround. So I'm getting info delivered to my laptop / phone / tablet in a timely fashion.

Obviously, I cannot guarantee that this will work for any configuration / for any specific goal -- but here is what I have tested myself. If you see what can be improved / corrected -- please, let me know!

## The problem
From time to time, I run my script, say, written in Python, on a remote machine (via these standard `qsub`, etc.). The problem is, quite often it takes pretty long time to run -- and I want to know what's going on:
- at the very least, I want to know when my computation proceeds to some next logical step (say, runs the next script);
- ideally, I want to be able to recieve some notifications from within the code -- if something really important happens. If possible, to my mobile phone. For example, last time my code was looking for counter-examples to some hypotheses, and it was important to notify me about the cases found, so I could start looking into these in parallel.

**A straightforward approach:** log in via ssh and check what's going on. It is relatively convenient, but still takes too much time: a word in the command line, paste the password, press a button on the mobile phone for the two-factor authentication, then at least one command to see if there is any output. Plus it takes some time to connect. 

**A standard solution:** there is [one](https://www.palmetto.clemson.edu/palmetto/userguide_basic_usage.html) (search for the word "mail" on that page) with the email -- which, for some reason, never worked for me. I think I tried to use it on two different clusters, as a user, and never got a single email[^1]. Besides that, from what I understand, I could get an email only when the job starts, ends or aborts -- so this seemed pretty restrictive anyways.

I had several requirements for mysel:
- little to no third party software needed on the remote node (I'm not sure to what extent I can install stuff on the cluster);
- minimal additional libraries for the programming language I use (what if, e.g., there's no package for Julia?);
- **NO COMPLICATED SETUP** -- I just don't have time for this. Like, I was not ready to implement a separate piece of software to do the job.

I was thinking about the [matrix.org](https://matrix.org/faq/) client-server [API](https://matrix.org/docs/guides/client-server-api) to employ, since it is free and open source[^2]. But it looked like too much effort for just a nice-to-have thing. But then, all of a sudden, I have stumbled upon a solution from a completely unexpected side: [Telegram](https://telegram.org/faq#q-what-is-telegram-what-do-i-do-here). Yes, it's a messenger that claims to be ``super-fast, simple and free''. I use it a lot, and find it, indeed, pretty convenient -- but hey, I thought that all the messengers were not that different. Or were they?..

## Level zero: it's alive!
There is something that is called [Bots](https://telegram.org/faq#bots). Non-playing characters. Computer programs you can chat with. And the secret is that the barrier to entry seems extremely low. Apparently, you can set up a bot in 5 minutes (literally) -- and use it with virtually no additional code. Let me share my experience. If you want to try yourself -- note that I'm assuming you already have Telegram [app](https://telegram.org/apps) installed on your phone, tablet, desktop, albeit a Mac, Win, Linux -- or wherever you prefer to have it -- and have an account (authors [claim](https://telegram.org/faq#q-how-is-telegram-different-from-whatsapp) that it "is free and will stay free - no ads, no subscription fees, forever").

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

## Level one: talk to me
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


## Level two: text me if anything happens
Now, this is already pretty flexible. However, you might want you script to text you in the middle of execution. For example, if my script is looking for certain examples, it would be nice if it texted me "hey, look at the instance No. X!" and continued searching. 

### In Python
In Python it [can](https://medium.com/@ManHay_Hong/how-to-create-a-telegram-bot-and-send-messages-with-python-4cf314d9fa3e) be done as follows (again, don't forget to fill in your token and chat id):

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
### In Julia
I was working with Python, but if you happen use Julia instead, it also shouldn't be a problem. In Python this ```python import requests``` did not require any additional setup -- however, when I tested it for Julia `1.1.1` it took some magic to perform first. I needed to install the package `HTTP` first. Like, 
- go to your login shell on the cluster;
- run an interactive job with (following the [advice](https://medium.com/@sitewang/clemson-palmetto-julia-optimization-beginners-guide-effe190be703) I was able to google): 
```bash
$ qsub -I -l select=1:ncpus=8:mem=16gb,walltime=01:00:00
```
- from an interactive job shell (check that your login prompt has changed and you are not on a login node any more), run:
```bash
$ module add julia/1.1.1
$ julia -e'using Pkg; Pkg.add("HTTP")'
```

Then get back to the login shell (`exit`, as usual). I guess, you'll need to perform this operation once and then just use the `HTTP` package. That is, write down the function to send a message to yourself -- pretty much identical to the one in Python above. Say, to test the thing I sketched the following `tgtest.jl`:

```julia
using HTTP

function tg_sendtext(msg)
    bot_token = "<your-token>"
    bot_chatID = "<your-chat-id>"
    send_text = "https://api.telegram.org/bot$(bot_token)/sendMessage?chat_id=$(bot_chatID)&parse_mode=Markdown&text=$(msg)"
    return HTTP.request("GET",send_text)
end

tg_sendtext("Hello from Julia (computation node)")
```

The script can be sent to ~~heaven~~ the computation node with a standard `qsub` -- like, prepare the `tgtest.pbs` (assuming the script is in the `misc` directory under your home dir):

```bash
PBS -N TestTG
#PBS -l select=1:ncpus=1:mem=2gb,walltime=01:00:00

# load necessary modules
module add julia/1.1.1

cd /home/$USER/misc
julia ./tgtest.jl
```

and just `qsub ./tgtest.pbs` it. 

And so, with around 20 additional lines of code I have all this glory on my mobile phone:

<div style="width: 30%; margin: 0px auto;">
![chat screen](/assets/tgpost/tg_screenshot.png)
</div>


If you have any suggestions, notes or comments that you thing would be useful for the content -- please, [let](/contact/) me know!

[^1]: most probably, I was doing something wrong. Please, share your experience if you were able to make it work on our Palmetto cluster!
[^2]: still, if you happen to use it, I'd be interested to learn from your experience
