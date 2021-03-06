<!--
.. title: Installing a fully fledged, ready to use mail server on CentOS 6 with Postfix, PostgreSQL, Amavis, ClamAV, Spamassassin and Dovecot - Part 3
.. slug: mailserver-3
.. date: 2013/04/05 18:00:00
.. tags: unix, postfix, amavis
-->

## Chapter 3 - Milters, the trusty workers

Welcome to the third part of our mailserver setup saga.

If you've missed [chapter 1](http://shisaa.be/postset/mailserver-1.html "First chapter of this mailserver series") or [chapter 2](http://shisaa.be/postset/mailserver-2.html "Second chapter of this mailserver series") I recommend reading them first.

In this episode, we will be looking a bit closer at our so called "Milters" or "Mail Filters".
Especially the ones who will protect us from evil virusses and floods of spam.

First, the outline of this chapter:

* Install and setup Amavis
* Configure Postfix to let Amavis do some heavy check lifting
* Install and configure ClamAV and make it run with Amavis
* Install and configure Spamassasin and make it run with Amavas

Lets dive in!

### Amavis

First lets talk about this "Amavis" program, what is it and what does it do...

Amavis or more better AMaViS stands for "A Mail Virus Scanner", it, well, scans for viruses and checks for spam.
Amavis is a very powerful interface between Postfix and third-party programs. Those third-party applications are most commonly *ClamAV* for virus scanning and *Spamassassin* for spam filtering.

In our factory, Amavis would be a separate small conveyor belt installed next to our big Postfix belt.
At that conveyor belt are many workers specially trained to check messages for viruses and spam.

That's actually all you need to know before installing...so lets go ahead and install it!

By default your CentOS 6 installation does not carry a recent binary package for that in its basic repositories.
So you will have to add one:

	:::bash
	$ rpm -Uvh http://apt.sw.be/redhat/el6/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.2-2.el6.rf.x86_64.rpm

And install it:

	:::bash
	$ yum install amavisd-new

You will notice that it installs a bunch of extra Perl modules needed to run Amavis. It also installed Spamassassin by default.
To make our setup complete, we also need ClamAV, so lets go ahead and install this one too:

	:::bash
	$ yum install clamd

Good, so now we have our three new programs installed, lets start by configuring our interface, Amavis.
The main configuration file for Amavis is located at */etc/amavisd.conf*. Open up that file.

This configuration file is quite daunting, so lets just configure it piece by piece, shall we?

Lets first check to see if anti-virus and spam filtering are enabled. Check if the two lines that start with *@bypass_spam_checks* and *@bypass_virus_checks* are commented out.
If you uncomment them, spam and antivirus checking will not happen.

Then we have to set *$mydomain* parameter to your domainname.

Next up are the *$daemon_user* and the *$daemon_group* variables. These should match the user and group under which Amavis is allowed to run.
If all is good, both should have been created during the install of Amavis, but it never hurts to check:

	:::bash
	$ grep "amavis" /etc/passwd

If it returns the user, everything went well, now lets check for the group:

	:::bash
	$ grep "amavis" /etc/group

It returned a group? Perfect! Then lets continue!

Now we can go back to our *amavisd.conf* file and uncomment the *$MYHOME* variable and also uncomment and set the *$myhostname* variable to read your FQDN: mail.example.com.

We also have to enable the newly installed ClamAV in our *amavisd.conf* file. There is already an entry for that. Search for ClamAV and uncomment these lines:

	:::bash
	$ ### http://www.clamav.net/
		['ClamAV-clamd',
			\&ask_daemon, ["CONTSCAN {}\n", "/var/run/clamav/clamd.sock"],
			qr/\bOK$/, qr/\bFOUND$/,
			qr/^.*?: (?!Infected Archive)(.*) FOUND$/ ],
        # # NOTE: run clamd under the same user as amavisd, or run it under its own
        # #   uid such as clamav, add user clamav to the amavis group, and then add
        # #   AllowSupplementaryGroups to clamd.conf;
        # # NOTE: match socket name (LocalSocket) in clamav.conf to the socket name in
        # #   this entry; when running chrooted one may prefer socket "$MYHOME/clamd".

Important to notice is the socket */var/run/clamav/clamd.sock*, this socket has to be the socket where ClamAV is really running.
To check this, open up the ClamAV configuration file at */etc/clamd.conf* and check the variable named *Localsocket*.
Make sure that both files have exactly the same path to that socket.

Also make sure that the path to that socket is accessible to the "clamav" user that was created when installing ClamAV.
So set the correct ownership:

	:::bash
	$ chown -R clamav:clamav /var/run/clamav

Then make sure that the user "clamav" is in the "amavis" group:

	:::bash
	$ usermod -G amavis clamav

Now amavis will use ClamAV as its primary virus scanner.

There are three other variables that you should make mental note of for now: *$max_servers*, *$notify_method* and *$forward_method*. We'll come back here later.

Good, that was a quick rush through the basics of the Amavis configuration, lets first tell Postfix about Amavis and make them work together before we delve a little bit deeper into our milters.
Open up our old friend the *master.cf* file from Postfix and add a new transport daemon to be spawned called *Amavisfeed*:

	:::bash
	$ amavisfeed unix -       -       n       -       2       lmtp
	    -o lmtp_data_done_timeout=1200
	    -o lmtp_send_xforward_command=yes

What did you just do?
Well, you created a so called *dedicated lmtp-client* that Postfix can use to communicate with Amavis.
The overwrites under it do the following:

	:::bash
	$ -o lmtp_data_done_timout=1200

This line sets a timeout limit in seconds for Postfix to wait to claim successful delivery. If a message is not delivered within this time limit, Postfix will give the message a "deferred" status and alert the sender.
We have this separate timeout setting, which is larger then the default Postfix timout, because we now add some new processes that the mail has to go through before it can be delivered.
The workers and the sides of our conveyor belt have just tripled and so has the amount of checking that needs to be done. We have to give these people some more time!

	:::bash 
	$ lmtp_send_xforward_command=yes

This tells Postfix to give Amavis the messages original IP-address and HELO command and not only the message itself.
With this extra information Amavis is able to do some extra checks to see if this message is genuine.

You should also note the *maxproc* setting on this transport, it reads "2". This should correspond to the *$max_servers* variable in the amavisd.conf file.

Now we have setup the basic integration for when Postfix receives mail. But there is one more important daemon to setup, the *reinjection smtp daemon*.
When Amavis is called to check a mail it will pick the mail up from the main Postfix conveyor belt and puts it on its own small conveyor belt.
The specially trained workers then go about there business as instructed by you, the Postmaster, in the amavisd.conf file.

After the checks are finished, the mail must go back on the conveyor belt so it can be delivered or trashed according to the milters findings.
The process of putting mail back onto the conveyor belt is called "reinjection". This has to be done by a separate SMTP daemon of which Postfix only accepts mail from Amavis.
A special entrance to the Postfix conveyor belt, so to say, only to be used by Amavis. Lets set up this new daemon:

	:::bash
	127.0.0.1:10025 inet n    -       n       -       -     smtpd
		-o content_filter=
		-o smtpd_delay_reject=no
		-o smtpd_client_restrictions=permit_mynetworks,reject
		-o smtpd_helo_restrictions=
		-o smtpd_sender_restrictions=
		-o smtpd_recipient_restrictions=permit_mynetworks,reject
		-o smtpd_data_restrictions=reject_unauth_pipelining
		-o smtpd_end_of_data_restrictions=
		-o smtpd_restriction_classes=
		-o mynetworks=127.0.0.0/8
		-o smtpd_error_sleep_time=0
		-o smtpd_soft_error_limit=1001
		-o smtpd_hard_error_limit=1000
		-o smtpd_client_connection_count_limit=0
		-o smtpd_client_connection_rate_limit=0
		-o receive_override_options=no_header_body_checks,no_unknown_recipient_checks,no_milters,no_address_mappings
		-o local_header_rewrite_clients=
		-o smtpd_milters=
		-o local_recipient_maps=
		-o relay_recipient_maps=

Waah! What's all this? 

Well, its nothing more then a new SMTP daemon, running on your localhost ip on port 10025 (remeber my mental note about $notify_method and $forward method?).
It has as many as 20 overwrites under it, all are used to overwrite default settings you may have set for your normal Postfix SMTP deamon.
You will recognize many lines from chapter 1 and some new lines that may look unfamiliar to you.

We could go about and explain each line in more detail but that unnecessary this time. All you need to remeber is that this deamon is used only by Amavis to send mail back to Postfix.
Any mail coming out of Amavis has the correct labeling and we can assume that these messages are save to re-inject or to put back on our conveyor belt.
To do more checking on this deamon would be redundant and would only consume more system resources.

Save that *master.cf* file and stop your Postfix daemon (*don't restart it*). We first have to make sure that our Spamassassin and ClamAV daemons are running and setup to run by default.
First start the Spamassassin, ClamAV and Amavis daemon:

	:::bash
	$ /etc/init.d/spamassassin start
	$ /etc/init.d/clamd start
	$ /etc/init.d/amavisd start

Then make sure that after reboot all daemons are automatically started:

	:::bash
	$ chkconfig spamassassin on
	$ chkconfig clamd on
	$ chkconfig amavisd on

Now you can start the Postfix daemon:

	:::bash
	$ /etc/init.d/postfix start

If all is well, we now should have basic spam and virus protection up and running!
Check you maillog, it should say all kinds of neat messages about Amavis decoders, AV Scanners, etc.
Also check if Postfix is still accepting mail by putting a follow tail on your maillog and sending mail from external to one of your mailboxes:

	:::bash
	$ tail -f /var/log/maillog

When you hit send in your external mail program, the maillog should show some lines ending with "(delivered to maildir)".

Now lets do some testing before we continue. Lets see if the Amavis service is actually listening:

	:::bash
	$ telnet localhost 10024

This should bring up a telnet interface stating that amavisd-new service is ready.
Send a "ehlo" command on that telnet interface so see if you get output:

	:::bash
	$ ehlo localhost

If it prints a bunch of lines starting with "250" you are in the clear. Type "quit" to exit the telnet.
Second we want to test if our SMTP daemon we created above, the daemon used for re-injection, is also up and running:

	:::bash
	$ telnet localhost 10025

This should start a new telnet interface with a line reading something like "220 mail.foo.com".
Again give the ehlo command to see if we get something:

	:::bash
	$ ehlo localhost

Another row of 250 lines? Good! Use quit to exit this telnet and lets move on!

Now its time to tell Postfix to pass all incoming mail to the small conveyor belt of our Amavis workers.
Open up your main.cf file and at the end, where you inserted the PostgreSQL lines in the previous chapter, add this simple line:

	:::bash
	$ content_filter = amavisfeed:[127.0.0.1]:10024

And there you have it, Postfix will now allow Amavis to pull through all the mail!

Now its time for sending some virusses and spam to our server, the proof of the pudding is in the eating so to speak.
First, lets see if our Amavis milters catch this virus test string created by the folks over at eircar.org.
Send the following string as a plain text message from an external mailserver to your mailserver and check your maillog after sending:

	X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*

This string is save to send and does not contain a virus, read more about this at [Eicar](http://www.eicar.org/86-0-Intended-use.html).
This merely triggers Amavis's and in turn ClamAV's warning light and it will alert you of a possible virus in the maillog.
Your maillog will contain a bunch of new lines about this message where one will say something like:

	:::bash
	$ mail amavis[5112]: (05112-03) Blocked INFECTED (Eicar-Test-Signature)

If you see this, congratulations, Amavis, ClamAV and Postfix are working together nicely! 

Now lets do a spam test. The website of Spamassassin delivers us with a same type of string that will fire your setup's spam warning lights.
Again, send a plain text email with the following string:

	XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X

Now check your maillog, again a bunch of new lines and one of them should say:

	:::bash
	$ mail amavis[5113]: (05113-03) Blocked SPAM.

Good, everything seems up and running!

Now in your maillog you may also have noticed that it tries to alert you of this virus or spam via the address virusalert@yourdomain.com.
You can of course set this to any address you like by editing the amavisd.conf file.
Search for the variable "$virus_admin" and change the "virusalert" part into any other mailbox or alias that you wish to receive notifications on.

Whats next?

Well, the world of viruses and spam is changing rapidly, every day. To have a good protection it is thus necessary to also keep our systems up to date.
First lets auto-update Spamassassin. Spamassassin already prepared a cronfile for us that updates its spam rules.
The cronfile is located in */usr/share/spamassassin/sa-update.cron*. During install, Spamassassin already added a cronjob for this file in */etc/cron.d/sa-update*.
Just open the last file and check if the line is not commented:

	:::bash
	$ 10 4 * * * root /usr/share/spamassassin/sa-update.cron 2>&1 | tee -a /var/log/sa-update.log

Once a day this will update the spam rules via cron if you have enabled it in your crontab file.
To enable this in our crontab file, op this file at "/etc/crontab" and make sure that our hourly, daily, weekly and monthly scripts are there.
By default this file is empty, only with a comment header telling you about how to use the crontab. You can enable the different cronjobs by adding this to your crontab:

	:::bash
	$ 01 * * * * root run-parts /etc/cron.hourly 
	  02 4 * * * root run-parts /etc/cron.daily 
	  22 4 * * 0 root run-parts /etc/cron.weekly 
	  42 4 1 * * root run-parts /etc/cron.monthly

Save this file. Now all the cronjoobs in the various cron.* directories are executed at their set interval.
Now make sure the cron daemon is running:

	:::bash
	$ /etc/init.d/crond start

And set it to run automatically after reboot:

	:::bash
	$ chkconfig crond on

After the first cron has run, you can check the update log file at */var/log/sa-update.log* to see if everything went well.
You can always call *sa-update* manually and restart Spamassassin by issuing the following command:

	:::bash
	$ sa-update && /etc/init.d/spamassassin restart

The next one up is ClamAV's virus definition files. Also here ClamAV already installed a cronjob for us that updates the definition database one time each day.
It uses a shellscript called "freshclam" which can be found at */etc/cron.daily/freshclam* to do the job. Since you already configured your cronjobs to run, everything is setup.
Also here we can check a log file to see if it is being updated. This file is located here: */var/log/clamav/freshclam.log*.
This update can also be triggered manually by issuing the following command:

	:::bash
	$ freshclam && /etc/init.d/clamd restart

Okay, we are through! We now have full virus and spam protection up and running.
This chapter wasn't to bad, right?

On to [chapter 4](http://shisaa.be/postset/mailserver-4.html "Fourth chapter of this mailserver series"), where we will setup Dovecot which will connect our beautiful factory to the outside world!

And as always, thanks for reading!
