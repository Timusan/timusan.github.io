<!--
.. title: Installing a fully fledged, ready to use mail server on CentOS 6 with Postfix, PostgreSQL, Amavis, ClamAV, Spamassassin and Dovecot - Part 5
.. slug: mailserver-5
.. date: 2013/06/08 20:30:00
.. tags: unix, postfix, dovecot
-->

## Chapter 5 - Dovecot continued: our frontdesk in more detail

Okay, I actually wanted this saga to be a four chapter series and I kinda did keep it at that, but there where a few things I noticed myself configuring inside my own shisaa.jp mailserver that I did not mention in the previous four chapters. Maybe I wanted to keep the previous chapters short...or...well...less long, or I was just being a lazy bastard.

Which ever being the case, lets see Dovecot in a little bit more detail.

### Time moved backwards

First, lets mingle with time. Or to be more precise, lets *fix* time....the time on our server that is. What does this have to do with Dovecot?
Well, if you are using Dovecot and you never got *"Time moved backwards"* as an error in your maillog, then good for you!
It either means that you have configured the time correctly on your server, or that you just are one lucky falla (or gal).
If you *do* have experienced this, you will know that this little error kills the Dovecot daemon and that you are left without a frontdesk to ask for your mail.

So what does this error mean and why does Dovecot crash?

*"Time moved backwards"* means literally what it says: The time on your server suddenly moved back in time.

How?

This comes down to time synchronization and how you set this up. 
If you have no time sync whats so ever, then your are fine. And with fine I mean fine for this error, actually your servers time is probably behind the actual time, you just don't know it.

But if you are getting this error, it means you *have* setup time synchronization (yeah!), but you did it the wrong way (oh...).
You have most probably setup time to sync itself with *ntpd* using a periodical cronjob that calls *ntpdate*.
This will check the time against a NTP server every time the cronjob runs and immediately adjusts your servers system clock to match.
Because this adjusting is done instantly, the clock will skip time (forwards or backwards). This may only be a matter of a few seconds, but the time will shift harshly non the less.
This "jumping" of the time is what will crash Dovecot.

Before we go into the simple solution to this problem, we have to ask if this is a bug in Dovecot or if this is desired behavior.
To answer that we have to look at the implications that can come when time moves suddenly.

For one, the files that are created in your maildir(s) rely on a timestamp for their uniqueness. If two timestamps would overlap, this could resolve in data loss.

Second, timestamps in Unix are used in memory constantly, not only by Dovecot. Shifting in time could resolve in a possible unstable system.

Is crashing desired behavior then?
Well, not really, that's why as of version 2.x, Dovecot doesn't crash but it still stops the SMTP and POP3 daemon.
This will prevent nasty things happening to your maildir files. It won't, however, save you from other potential instabilities that could happen in your system.

The solution to this sudden jump in time is quite simple: Don't jump, stretch. If your clock is behind, speed up the time until it reaches the correct time. If time is in the future, slow it down.

Makes sense, right?

To setup this stretching is even simpler. You still use NTP, but the correct way.
First make sure that you have NTP installed:

	:::bash
	$ yum install ntp

Next make it run on boot:

	:::bash
	$ chkconfig ntpd on

Before starting the daemon we have to setup which timeserver we want to synchronize with.
Open the *ntp* configuration file located at */etc/ntp.conf*.
In here, setup the timezone server you which to use. To get a list of servers visit the [NTP Pool Project](http://www.pool.ntp.org/zone/@ "NTP Pool Project website").

	:::bash
	$ server 0.your-favorite-timezone-server.org

Now save the file and simply start the daemon:

	:::bash
	$ /etc/init.d/ntpd start

That's it, now your server time will be updated gracefully.

### Filtering mail

Maybe you want to put mail from specific addresses in specific mailfolders, you want to put messages marked as spam in its own container or you want to split work and private related email from the same address, which ever being the case, you need filtering.

And whats more important, you want to filter server side.

Why?

If you check your mail on multiple devices, you don't want to setup the filter rules on each client. You want to manage your filtering on the server and watch the magic happen automatically on each machine you use.

The combination of Postfix and Dovecot is ideal for this task.

If you have followed [my](http://shisaa.jp/postset/mailserver-1.html "First chapter of this mailserver series") [previous](http://shisaa.jp/postset/mailserver-2.html "Second chapter of this mailserver series") [four](http://shisaa.jp/postset/mailserver-3.html "Third chapter of this mailserver series") [chapters](http://shisaa.jp/postset/mailserver-4.html "Fourth chapter of this mailserver series") and setup your own mailserver, then currently Dovecot is only used to check mail and authenticate, all through the IMAP protocol.
The actual delivery is done by Postfix itself using the Postfix LMTP delivery mechanism. This is a simple mechanism that, well, just delivers mail and does nothing else.
To setup server side filtering we have to plugin Dovecot's *Local Delivery Agent* or *LDA* into our workflow. Substituting Postfix LMTP for Dovecot LDA gives us two main advantages:

* By letting Dovecot deliver the mail to the users mailfolder the filtering is done at the correct timing in the delivery workflow, while the mail is in transit, before it is delivered.
* By using this LDA we can directly tap into Dovecot's powerful filtering plugin language and setup some fancy rules with ease.

So lets get our hands dirty and introduce a new workstation in our factory!

First we start with setting up the Dovecot LDA. To do this we have to introduce a new protocol in the Dovecot configuration file. Add the LDA protocol to *dovecot.conf*:

	:::bash
	protocol lda {
	  mail_plugins = $mail_plugins
    }

This will initiate the LDA mechanism and load in any plugins we define. We will need this later down the road.
Next we need to define 2 important variables for Dovecot to be able to act as a delivery agent. Add these to the same file:

	:::bash
	postmaster_address = tim@shisaa.jp
	hostname = mail.shisaa.jp

Since Dovecot will do the delivery, it will now also do the rejection handling if something is wrong (instead of Postfix). When it rejects mail it needs to know where to sent a notification to.
The hostname is used for including in the message headers.

Now we have to input some familiar settings we used to make Dovecot communicate with Postfix. But this time, we need them for the user lookups that the LDA will have to do.
To be more specific, we need to redeclare the *userdb* and the *auth-userdb* declarations.
First the *userdb*:

	:::bash
	userdb {
		driver = sql
		args = /etc/dovecot/dovecot-sql.conf
	}

**Caution**: leave your original *userdb* declaration intact. For Postfix we have setup the user lookup to *driver=prefetch* because we had enough information from the *passdb* query and we did not want to travel to the database twice. For the LDA, however, this is different. The lookup cannot work with the driver set to *prefetch*, we will have to do a separate query. So now you should have two *userdb* entries like so:

	:::bash
	userdb {
		driver = prefetch
	}
	userdb {
		driver = sql
		args = /etc/dovecot/dovecot-sql.conf
	}

Postfix will take the *prefetch* usersb and the LDA will take the second, because it fails using the first.
The next thing is to declare a user lookup query in our *dovecot-sql.conf*. Open this file and add the following query:

	:::bash
	user_query = SELECT '/home/mail/'||maildir as home, uid, gid, email FROM users WHERE email = '%u'

This looks quite similar to the *password_query* we defined in chapter 4, and it is actually a bit redundant, but Dovecot LDA does simply not accept prefetch.
There is one pitfall you have to look out for: the *maildir* or *home* if you will. As you can see we do the same concatenation as we did in the last chapter, but this time I have added an extra slash.
So *'/home/mail'* becomes *'/home/mail/'*. I contrast to the Postfix LMTP, the LDA needs the extra slash at the end. So you could put an extra slash in your database entries, or just put it one time in the lookup queries and be done with it.

**Don't forget to also put this extra slash in your password_query as well!**

Another thing you may notice is that the concatenated *home* in the *user_query* does **not** start with *maildir:*. If you do so, Dovecot will complain about not supporting relative home directories.
But if you don't put it their, the LDA will start complaining that it does not know in which format the mail directory is laid out. To fix this we have to return to our *dovecot.conf* and add the following line:

	:::bash
	mail_location = Maildir:~/

This will tell Dovecot that our mail directories are in *Maildir* format. The lookup should succeed now.

The last thing we have to edit in our *dovecot.conf* is the unix listener for the auth_userdb socket.
Again, you already did this one chapter ago, but this was for the Postfix SMTP daemon.

**You also need to keep the original entry!**. Add the following entry:

	:::bash
	service auth {
		unix_listener auth-userdb {
		mode = 0600
		user = mailreader
		}
	}

This will add another auth-userdb socket, besides the Postfix one, that the LDA will use to do the user lookups.
If you keep the mode set to 600, you don't need to add a group, only our famous *mailreader* user is okay.

Save the file and close it up. We are finished here.

What's next?

We need to tell Postfix about the new player in town, so crack open your master.cf and add the following service:

	:::bash
	dovecot   unix   -   n   n   -   -   pipe
	  flags=DRhu user=mailreader:mail argv=/usr/libexec/dovecot/deliver -f ${sender} -d ${recipient}

This will enable the LDA service and use Postfix *pipe* mechanism to pass mails that are ready to be delivered to the Dovecot LDA.

Whats a *pipe* mechanism?

In Postfix, the *pipe* mechanism or *pipe* daemon is a way for postfix to deliver mail to a command line program.
In our case, we pipe the mail to an external program called *deliver*, which you can also run from the prompt.

The *pipe* mechanism comes with a few attributes you can set. To begin, we have the *flags*. With *flags* you can modify messages before they are handed over to the piped program.
In our case we say *DRhu*:

* D: Will include a *Deliver-To* in the message header
* R: Will include a *Return-Path* in the message header
* h: Will lowercase everything right of the @ sign (or right of the most right @ sign)
* u: Will lowercase everything left of the @ sign (or left of the most right @ sign)

So basically we include some extra information and make sure the addresses are all lower cased.

Next we specify as which user:group that the *deliver* command has to be run.

The *argv=* part tells Postfix which program we actually want to call, being the Dovecot LDA.

The *-f* and *-d* flags at the end are flags belonging to the *deliver* program itself.
With *-f* you can define the *Envelope sender*, with *-d* you define the username.
After each flag you can find a placeholder, these are placeholders defined by the Postfix pipe.

Are we done?

Almost! The only thing you have to do now is tell Postfix to actually *use* the LDA service. To do that we have to alter our *virtual* transport (which is the standard Postfix LMTP) and set that to the name of the service, in this case *dovecot*. Since we are using a fancy database lookup system, we will have to alter the entry in the *transport* table of our mail database. Change the transport from *virtual* to *dovecot* in the record that accompanies your domain (in most cases you will have just one entry in that table).

Just restart both *Postfix* and *Dovecot* and you are done!

If all is well, your mail should still be working and nothing should have changed (that's a good thing).
But if you check your famous *maillog* file, you will see entries about *dovecot: lda* putting stuff in your mail directory.

And now...the actual filtering.

For filtering, Dovecot has a nice plugin called *Sieve* which uses the (surprise) *Sieve Language* for scripting together filtering rules.
To get up and running, we need to install the Sieve interpreter and Dovecot plugin:

	:::bash
	yum install dovecot-pigeonhole

Yes...its name is Pigeonhole, I cannot help it.
Now we have to load the plugin for our LDA. Open the *dovecot.conf* and go to the freshly inserted *protocol lda* block and append the *sieve* plugin:

	:::bash	
	protocol lda {
		mail_plugins = $mail_plugins sieve
	}

And restart Dovecot.

Next you need to go the *plugin* declaration block where you defined your different *autocreate* and *autosubscribed* behaviors in chapter 4.
Here we can do the basic configuration for the *sieve* plugin. Many hosted mail providers that run of a Dovecot service provide the ability for users to upload their own custom sieve scripts.
If you want to do so, you can declair the *sieve=* variable, which most of the time would be set like so:

	:::bash
	sieve=~/

This tells *sieve* that user script can be found in the users home directory.
For the scope of this post I'm only interested in setting up a global script for all users. The *global_script_path* does this for us:

	:::bash
	global_script_path = /home/mail/default.sieve

Now for the fun part, make your *default.sieve* file and lets start making a rule!
In the beginning of your Sieve script you have to define which modules you want to load in.
I first want to file messages that are marked *spam* by Amavis into a spam mail folder. So I only need the *fileinto* plugin which handles moving messages around.
Start your script:

	:::sieve
	require "fileinto";

Then for the rule:

	:::sieve
	if header :contains "X-Spam-Flag" "YES" {
		fileinto "Spam";
	}

Bam! That's quite readable and simple, won't you say? If the header has it spam flag set to "YES" file it into spam!
Save this file into the defined location and make sure it is readable by Dovecot, so in our setup that would be:

	:::bash
	chown mailreader:mail /home/mail/default.sieve

Now remember, the *"Spam"* in this script corresponds to a folder that you made with Dovecot. If you have not made a *Spam* folder, go and make one now, checkout [chapter four](http://shisaa.jp/postset/mailserver-4.html "Fourth chapter of this mailserver series") for details on how to do this.

Next you want to put messages from a specific someone into its own folder? Lets do that then!
For this we need "fileinto" and also a plugin to read out the message envelope.

	:::sieve
	require ["fileinto","envelope"];
	if envelope "from" "tim@shisaa.jp" {
		fileinto "Shisaa";
	}


And what about that vacation responder? Also, piece of cake:

	:::sieve
	require "vacation";
	:days 1
	:subject "I'm on vacation!"
	:addresses ["tim@shisaa.jp,postmaster@shisaa.jp"]
    "I'm on vacation, you can join me or wait until I'm back.
    Cheers
    Tim";

A little bit more explenation may be in place. *days 1* simply tells Sieve to only send the same person the out-of-office reply once every day.
The rest is pretty straight forward. You input the subject, the addresses and the message itself.
And of course you can put everything together in one big Sieve script:

	:::bash
	require ["fileinto","envelope","vacation"]
	if header :contains "X-Spam-Flag" "YES" {
		fileinto "Spam";
	}
	if envelope "from" "tim@shisaa.jp" {
		fileinto "Shisaa";
	}
	:days 1
	:subject "I'm on vacation!"
	:addresses ["tim@shisaa.jp,postmaster@shisaa.jp"]
    "I'm on vacation, you can join me or wait until I'm back.
    Cheers
    Tim";

You don't need to restart Dovecot after altering a Sieve script.

Okay, we have gone out and had a little deeper chat with our frontdesk Dovecot. Quite a nice and capable person don't you think?

If you run into any trouble, be sure to always check your maillog , for it will tell you quite clear (most of the time) whats going on.

Maybe a small tip for debugging: If you want more verbose output from the whole LDA process (including Sieve) you can put this in the *protocol lda* block:

	:::bash
	mail_debug=yes

Now go out and learn more about what you can do with Sieve and harvest that knowledge!

And as always, thanks for reading!
