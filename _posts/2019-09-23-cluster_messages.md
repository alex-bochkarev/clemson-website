---
layout: post
title: Text and derive - getting notifications from the cluster
categories: [tools]
tags: [cluster, linux]
menuid: "Tools"
---

Some time ago I started to work with the computing cluster again -- running python scripts on remote nodes. So I got back to that problem of receiving information from my program in a timely fashion. I think I have found a super simple and super efficient set up, so I'm getting the info from my scripts delivered to my laptop / phone / tablet. Couldn't help sharing.

Obviously, I cannot guarantee that this will work for any configuration / for any specific goal -- but here is what I have tested myself. If you'll have any suggestions or notes that you think would be beneficial for the note -- please, send me an email.

## The problem
From time to time I run my script, say, written in Python, on a remote machine (via this [standard](https://www.palmetto.clemson.edu/palmetto/userguide_basic_usage.html#job-submission-and-control-on-palmetto) `qsub` magic). The problem is, quite often it takes pretty long time to run -- and I want to know what's going on:
- at the very least, I want to know when my computation proceeds to some next logical step (say, runs the next script);
- ideally, I want to be able to recieve some notifications from within the code -- if something really important happens. For example, last time my code was looking for counter-examples to some hypotheses, and it was important to notify me about the cases found, so I could start looking into these while my code were still working;
- moreover, I'd prefer to have it with little to no additional 3-rd party software and definitely **no complicated setup** (not sure how simple would it be to install things on the cluster).


A straightforward approach would be to login via `ssh` and just check what's going on. There is also a standard [feature](https://www.palmetto.clemson.edu/palmetto/userguide_basic_usage.html#pbs-job-options) for getting an email when the job starts, ends and/or gets interrupted (see `-m` and `-M` options for the `qsub` command). Although, this was not exactly what I wanted[^1]. 

I have just stumbled upon a solution that turned out to be quite simple -- the one using [Telegram](https://telegram.org/faq#q-what-is-telegram-what-do-i-do-here). Yes, it's a messenger that claims to be ``super-fast, simple and free''. I use it a lot, and find it, indeed, pretty convenient -- but hey, I thought all the messengers were not that different. Or were they?..

## Level zero: ``it's alive!''
There is something that is called [Bots](https://telegram.org/faq#bots). Non-player characters. Computer programs you can chat with. And the secret is that the barrier to entry seems extremely low. Apparently, you can set up a bot in 5 minutes (literally) -- and use it with virtually no additional code. Let me share my experience. I'm assuming you already have the Telegram [app](https://telegram.org/apps) installed on your phone / tablet / desktop with any popular system, and have an account (authors [claim](https://telegram.org/faq#q-how-is-telegram-different-from-whatsapp) that it "is free and will stay free - no ads, no subscription fees, forever").

First, let us create a bot. Go to the app, find a "user" called [BotFather](https://t.me/botfather) -- this is actually a bot for creating bots (a recursive bot, yes. Note that if you have an app installed on the laptop -- you can just click the previous link, it will try to open in the telegram). In a chat window, let us tell it we'd like a bot:
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

Now, we need to do the only piece of magic in the whole process. Find your bot by the username you have just given it (for example, I have typed `@ab_wintermute_bot` into the "Search" field in the Telegram app). Open a chat and tell it `/start`. I also needed to tell something else -- like, `test`. The thing will listen but will not respond (obviously, since we haven't implemented anyone who could respond). We need this to obtain a so called "chat ID": now, open up your browser and open the URL: `https://api.telegram.org/botAAA:BBB/getUpdates`, replacing *AAA:BBB* with your "token ID" given by the `BotFather`. Look for the `chat id` and write it down somewhere:

<div style="width:50%; margin: 0px auto;">
![Chat ID](/assets/tgpost/chat_id.png)
</div>

Essentially, this is everything we need to start chatting with our remote scripts.

## Level one: ``talk to me''

First, let us create a short script to send messages to ourselves (it will listen `stdin` and just re-send everything to you). Open up your favorite text editor and copy the following, replacing \<your-token\> and \<your-chat-id\> with the values you obtained in the previous section:

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
python ./my_awesome_script.py > result.log
echo "Done with the calculation! Please log in and check" | ./tgsend.sh
```

At least now we have notifications about start and end of the job, delivered to your mobile phone. **Note:** it won't tell us anything if the job was killed (memory limit exceeded, walltime limit exceeded, etc.). For me it was a minor issue -- but if you know an easy fix, please let me know. As a workaround, if it is critically important, we could always ask the script to ping us, say, every half an hour with "I'm alive" kind of message.


## Level two: ``text me if anything happens''

Now, this is already pretty flexible. However, you might want your script to text you in the middle of execution. For example, if my script is looking for certain examples, it would be nice if it texted me "hey, look at the instance No. X!" and continued searching. 

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

Then, whenever you need to text yourself within your code, just do the obvious:
```python
tg_sendtext("Counter-example found: instance no. {}".format(instance_no))
```
### In Julia
I was working with Python, but if you happen use Julia instead, it also shouldn't be a problem. In Python this ```python import requests``` did not require any additional setup -- however, when I tested it for Julia `1.1.1` it took some effort. I needed to install the package `HTTP` first. Like, 
- go to your login shell on the cluster;
- run an interactive job with (following an [advice](https://medium.com/@sitewang/clemson-palmetto-julia-optimization-beginners-guide-effe190be703) I was able to google): 
```bash
$ qsub -I -l select=1:ncpus=8:mem=16gb,walltime=01:00:00
```
- from an interactive job shell (check that your login prompt has changed and you are not on a login node any more), run[^3]:
```bash
$ module add julia/1.1.1
$ julia -e'using Pkg; Pkg.add("HTTP")'
```

Then get back to the login shell (`exit`, as usual). I guess, you'll need to perform this operation once and then just use the `HTTP` package. That is, write down a function to send a message to yourself -- pretty much identical to the one in Python above. Say, to test the thing I sketched the following `tgtest.jl`:

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

The script can be sent to the computation node with a standard `qsub` -- like, prepare the `tgtest.pbs` (assuming the script is in the `misc` directory under your home dir):

```bash
PBS -N TestTG
#PBS -l select=1:ncpus=1:mem=2gb,walltime=01:00:00

# load necessary modules
module add julia/1.1.1

cd /home/$USER/misc
julia ./tgtest.jl
```

and just `qsub ./tgtest.pbs` it. 

And so, with around 20 additional lines of code we have all this glory on the go:

<div style="width: 30%; margin: 0px auto;">
![chat screen](/assets/tgpost/tg_screenshot.png)
</div>


## Couple of concluding notes
If you have any suggestions, notes or comments that you think would be useful -- please, [let](/contact/) me know!

At first, I was thinking about the [matrix.org](https://matrix.org/faq/) client-server [API](https://matrix.org/docs/guides/client-server-api) to employ, since it is free/open source and decentralized (so in theory it allows not to be locked in with any third-party service). But it looked like too much effort for just a nice-to-have thing. It worked with `curl`, but the messenger interface seemed to be significantly less smooth (e.g., I had some problems with notifications on Android and, for some reason, did not have a reliable messaging from computation nodes). I'd be glad to learn from your experience if you were able to make it work efficiently.

Another note is that I was not considering security/privacy issues at all -- you might want to read the docs more carefully if you work with this anywere near ``production'' (whatever this might mean to you).

[^1]: besides, for some reason this email thing never worked for me. Most probably, I was just doing something wrong. There might be even a good standard solution for the problem I am just not aware of. Please, share your experience if you were able to make it work, especially on our Palmetto cluster
[^2]: still, if you happen to use it, I'd be interested to learn from your experience
[^3]: for some magical reasons, whenever I tried to run interactive Julia and then do `Pkg.add` from it, the whole thing hanged. My guess is that doing `julia -e` from bash instead might be important
