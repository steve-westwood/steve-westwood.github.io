---
layout: post
title: "Scheduling Tasks with Cron on Mac"
date: 2019-06-06
---
The other day I had one of those _tasks_ that occasionally pop-up in the life of a developer where I had to perform a one-off action to one of my company's applications. I say the other _day_ but of course I mean the other _night_, because the action in question had to happen at 3am. Not for all the will in the world am I getting up at 3am to run a shell script. Thankfully I don't have to :sweat_smile:

## Cron for MacOSX

So the solution I'm looking for is to schedule running a shell script using my development laptop (a Mac with OSX on). Obviously there is a strong argument for NOT launching these type of jobs on a development computer, there should be a more robust server bound solution for running scheduled tasks, and I completely agree. However the reality is sometimes it's easier to do something ad-hoc, especially if there is no existing process for one-off scheduled tasks in your existing infrastructure.

In these situations **[Cron](https://en.wikipedia.org/wiki/Cron)** is the perfect solution.  You can leave you development computer on and the task will be fired when you configure it to do so. So, how do configure a cron job to run on a Mac? 

## Configuring a Cron Job on Mac

To open your current scheduled task list, tap the following command into your terminal:

`crontab -e`

This with open a `vim` editor with the list of jobs (for the current _user_). This should be empty unless you've created any cron jobs recently.

(If you're not familiar using _vim_ to edit files before, check out a vim cheat sheet like [this](https://vim.rtorr.com/) (for example).)

So let's add a new cron job. But first, a **gotcha**...

I came across [an issue whereby if I used any command line utilities in shell script](https://stackoverflow.com/questions/26480860/aws-not-working-working-from-cronjob), cron couldn't _find_ that utility, because it doesn't have access to the `$PATH` env var. So before you add any cron jobs to you're crontab file copy in your `$PATH` and `$SHELL` vars to the top of the file:

```(shell)
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Once that is done, let's add a job. 

Cron works by assigning a time and and excution path in the following format ([see here for extra details](https://crontab.guru/))

`[minute] [hour] [day of month] [month] [day of week] /pathto/shellscript`

Say we want to add cron job to execute a script on [2:14am August 29th](https://www.neatorama.com/neatogeek/2013/08/29/August-29th-Skynet-Becomes-Self-aware/) 

`14 02 29 8 * /skynet/becomes-self-aware.sh`

(`*` in this instance means `any` or ignore (sort of))

It's best to add the **full path** to the script as the tilde `~` home shortcut can cause side effects.

Once you've saved the file and exited (`:x` in vim) cron will tell you it's configuration has been updated.

`crontab: installing new crontab`

## Tell me more...

If you want to pass in arguments to the shell script just add them after the path to the script.

`14 02 29 8 * /skynet/become-self-aware.sh nukem --all`

## Logging

Logging out the `stdout` and `stderr` output from the file is a good idea, because cron logging is non-existent on mac. The following command will append both outputs to a `judgment-day.log`.

`14 02 29 8 * /skynet/become-self-aware.sh nukem --all >> /skynet/judgment-day.log 2>&1`

## Debugging

Debugging using the `logs` you've just created is good start, additionally cron will send you mail!

Enter `mail` into a terminal after you've expected the cron job to have ran, and you should see a list of mail (in this instance I've told cron to schedule a `echo 'hello from cron'` command).

```
Mail version 8.1 6/6/93.  Type ? for help.
"/var/mail/steve": 1 message 1 new
>N  1 steve@mac.l  Thu Jun  6 15:06  19/729   "Cron <steve@mac> "
?
```

Hit `<enter>` to read the mail.

```
Message 1:
From steve@mac.local  Thu Jun  6 15:06:01 2019
X-Original-To: steve
Delivered-To: steve@mac.local
From: steve@mac.local (Cron Daemon)
To: steve@mac.local
Subject: Cron <steve@mac> echo 'hello from cron'
X-Cron-Env: <SHELL=/bin/bash>
X-Cron-Env: <PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin>
X-Cron-Env: <LOGNAME=steve>
X-Cron-Env: <USER=steve>
X-Cron-Env: <HOME=/Users/steve>
Date: Thu,  6 Jun 2019 15:06:01 +0100 (BST)

hello from cron
```

As you can see there's some _basic_ details including when the cron job has been executed and superficial output. 

(For more details on the `mail` utility enter `man mail` into a terminal.)

## Conclusion

Cron is really useful utility for running scheduled tasks and it relatively simple to set up once you've navigated a few pitfuls (and manage to escape vim!)

One warning though, might be a good idea to delete one-off jobs once they are done, as they will happen exactly the same time next year if you've followed this guide!

Just `crontab -e` and delete the offending job then save the file.

Hope you have found this blog post useful!