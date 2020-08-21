<!--
.. title: Installing a fully fledged, ready to use mail server on CentOS 6 with Postfix, PostgreSQL, Amavis, ClamAV, Spamassassin and Dovecot - Part 4
.. slug: mailserver-4
.. date: 2013/04/20 10:30:00
.. tags: unix, postfix, dovecot
-->

## Chapter 4 - Dovecot, the friendly front desk

And so we arrive at the final big stop of our mailserver series.

If you've missed [chapter 1](http://shisaa.jp/postset/mailserver-1.html "First chapter of this mailserver series"), [chapter 2](http://shisaa.jp/postset/mailserver-2.html "Second chapter of this mailserver series") or [chapter 3](http://shisaa.jp/postset/mailserver-3.html "Third chapter of this mailserver series") I recommend reading them first.

First, the outline:

* Small introduction to Dovecot
* Introduction to IMAP and POP3
* Configure Dovecot to use IMAP
* Setup SSL/TLS inside Dovecot
* Hashing our users passwords with doveadm
* Setting up extra mail folder
* Connecting with an email client

So far we have talked about the heart of our mailserver, Postfix, we then saw how we can use a solid database as our back-end instead of using Unix user accounts and in the last chapter we saw how we could protect ourselves from viruses and spam. So far we have a smoothly running factory, receiving mail, checking it and if necessary storing it. But there is still one important part missing, it is all happening behind the closed doors of our factory floor and there is still no way for our customers, the owners of the various mailboxes, to actually receive and send their mail. Our factory does not have a front desk yet. So lets make one, shall we?

This front-desk is called, as you might have guessed, *Dovecot*. It calls itself an IMAP and POP3 mail server.
It takes care of the communication between the actual mail files stored on the hard disk drive and the email client the end-user will be using.
Dovecot of course also supports our database back-end we used to store the sensitive information about the user.

Now what is this *IMAP* and *POP3* that Dovecot supports, you might ask.

*IMAP* stands for *Internet Message Access Protocol*. The IMAP protocol makes it possible to have multiple users connect to the same mailbox and see the same messages. The mail stays on the server.
*POP3* on the other hand is a protocol that has to copy the mail to the users computer before it can be displayed.
When you want to see your mail on your computer and on your mobile device for instance, this can be confusing when using *POP3*, many *POP3* clients even delete the mail on the server when it is downloaded.
*IMAP* syncs between the server and the various clients that connect to the mailbox and it will only download a message and its attachments when asked for.
*POP3* has to download all the messages, including the attachments, before it can be display in your mail client.
Another big difference between the two is that *outgoing* mail, when using *IMAP*, is stored on the server and thus available on multiple devices. With POP3, its only stored on the local computer from which you sent it.

Knowing the main differences between the two protocols, I always prefer *IMAP*.
So lets go ahead and configure Dovecot the be able to read your mail using the *IMAP* protocol!

As always, we first have to install the Dovecot package. The CentALT version is quite recent, so you can just go ahead and install it.

	:::bash
	$ yum install dovecot

Dovecots main configuration exists in only one file located at */etc/dovecot/dovecot.conf*. Open it right up and lets dive in!
The first thing you want to set are the different protocols you want to support, since we decided on *IMAP* you can add a line at the end of the file:

	:::bash
	$ protocols = imap

I like to add everything at the bottom to keep it all together. I start with a comment line reading something like "#Added lines by me" and put my own configuration lines under that..
Then add the *base_dir*, the *instance_name* and the *login_greeting* variable. You find these variables commented out in the *dovecot.conf* file. Just copy them to the bottom and remove the hash.

Save the file and lets see what else we need to configure.

In chapter 1 we talked about being secure when it comes to data traveling between the server and the client computer.
We have to continue this work in Dovecot because not only will be exchanging passwords, we will also be syncing mail between the server and the client computer.
So lets first setup SSL/TLS inside dovecot. All the extra configuration files are made available in the */etc/dovecot/conf.d* directory.
Files starting with a two digit number and end with ".conf" are automatically loaded by the *dovecot.conf* file.
The other files are indented as a demonstration of what else you can do in the main *dovecot.conf* file and should not be altered.

I usually disable the automatic include of these files and just copy the needed variables to my main configuration. So in the *dovecot.conf* file, put a hash before the bottom two "!include" statements.
In this conf.d directory you will also find a file called "10-ssl.conf*. Open this one up and lets start moving some variables to our *dovecot.conf* file. First the enable SSL:

	:::bash
	$ ssl = yes

Next we see two very important lines, the *ssl_cert* en the *ssl_key* file. Dovecot already prepared a certificate and a private key for you to use.
Also copy these lines to the *dovecot.conf* file.
The only thing you must be sure of is that both files are only readable by root. So do a *ls -la* to check if this is so.
Now before we do anything else, fire up dovecot and lets try connecting:

	:::bash
	$ /etc/init.d/dovecot start

Then lets try to connect to localhost and ask for imaps:

	:::bash
	$ openssl s_client -connect 127.0.0.1:imaps

If you get an output containing a SSL key, the connection is successful.

When we connect with our mail client further up this chapter, you will notice it will complain about a self-signed certificate and that's correct.
We ARE using a self-signed certificate. As discussed in chapter 1 that is perfectly save, as long as the mail server only serves a handful of users and these user are educated in the risk of a MITM attack.
You could always go out and buy a certificate with the certificate mafia and you should if you are planning to setup a large, public mailserver. But for now, self-signed is the way to go.

In chapter 2 we enabled Postfix to work with PostgreSQL to retrieve the necessary mailbox data so Postfix could deliver the mail. This same data Dovecot will need to authenticate the users.
So we need to enable PostgreSQL support in Dovecot.

In the *conf.d* directory you will find a *auth-sql.conf.ext* file, this is an example file of how we can setup our database connection.
So we have to setup our PostgreSQL connection and make a new file with the correct lookup query. Setup the connection in the *dovecot.conf* file:

	:::bash
	$ passdb {
		driver = sql
		args = /etc/dovecot/dovecot-sql.conf
	 }
	 userdb {
		 driver = prefetch
	 }

You will notice a couple of things here, first we have two variables we setup: *passdb{}* and *userdb{}*.
The *passdb{}* is used for the authentication when the user logs in and the *userdb{}* is used to retrieve additional information about the connected user.

But since the only extra information we need in our setup is the location of the users mailbox, we can also use the "*passdb{}* for that and save a query every time a users connects.
To use *passdb{}* for this we have to tell Dovecot that the *userdb{}* uses the *prefetch* driver, meaning it will use the *passdb{}* query for its extra user data.
The driver is set to *sql* instead of *pgsql*, Dovecot only expects to see which kind of main driver it will be using. In the *dovecot-sql.conf* we will define it is PostgreSQL.

The actually connection string and the lookup query are stored in an separate file as described above.
So create a new file called *dovecot-sql.conf* in */etc/dovecot/* and put in your database details like so:

	:::bash
	$ driver = pgsql
	  connect = host=localhost dbname=mail user=mailreader password=some-password
	  default_pass_scheme = SHA512
	  password_query = SELECT email as user, password, 'maildir:/home/mail'||maildir as userdb_mail FROM users WHERE email = '%u'

First we define which subtype of the sql driver we will be using, in our case *pgsql* for PostgreSQL.
Then we have the connection string for, well, actually connecting to the database.
The third parameter sets the type of password encryption scheme that we used to store our password in the users table.
And finally we have the query itself, Dovecot recognizes *user* as the actual user, *password* as the column containing the hashed password and *userdb_mail* as the maildir.

Since our column names don't fully correspond with what Dovecot expects, we have to rewrite them as you can see in the query.
You can see that I prefix the maildir with *maildir:/home/mail*, this way I can tell Dovecot two things, the mailbox is in *Maildir* format and make the paths absolute so Dovecot can find them. At the time of writing Dovecot does not support relative maildir locations. There is also one placeholder in here, the *%u*. In the Dovecot SQL driver *%u* means the full username: username@tld.com. Since we stored our users with their full email, we can safely use this placeholder.

One more important thing to tell Dovecot is with which UID and GID it can handle the mail directories.
In chapter 2 we created the user *mailreader* which owns all the mail directories. We also decided not to include the GID and UID in the database as this would be overkill, so instead we hardcoded them into the *main.cf* of *Postfix*. We now have to do the same with the *dovecot.conf*. Add these parameters to your file:

	:::bash
	mail_uid = the-uid-of-your-mailreader-user
	mail_gid = the-gid-of-your-mailreader-user

**Update:** In newer Dovecot versions (after the writing of these posts) the default, minimal allowed UID lies above 200. Therefor, setting this variable to 200 will cause Dovecot to throw an error stating that this UID is not allowed. To fix this, set the *first_valid_uid* variable before your *mail_uid* inside your *dovecot.conf* (be sure to check that UID 200 is not already in use on your system!): 

	:::bash
	first_valid_uid = 200

Remember we talked about password hashing in chapter 2 and how we would use *doveadm* for creating our hashes? Lets do that right away!

In the above *dovecot-sql.conf* you will notice that the *default_pass_scheme* is set to SHA512, another very strong password hashing algorithm.
So lets create a SHA512 hashed password for our "foo" user, shall we?

You can query which kind of hashing schemes your Dovecot installation supports by using *doveadm*:

	:::bash
	$ doveadm pw -l

There is a big chance you will see SHA512 among them. Lets now hash our plain text password:

	:::bash
	$ doveadm pw -p plain_text_password -s sha512 -r 100

This will return a hashed string using the sha512 scheme and hashed 100 times. Before this string is the scheme name withing brackets {SHA512}, *dont remove this*!
Insert this string into your database as the password for the user "foo".

	:::postgresql
	$ UPDATE users SET password = 'your-hashed-string" WHERE email = 'foo@domain.com";

Before you can login with your mailclient, be sure to enable *PLAIN* text login. Add the support for this in your *dovecot.conf* file:

	:::bash
	$ disable_plaintext_auth = no
	  auth_mechanisms = plain

And now...you should be able to connect with an external mail client to your server. Setup your client to connect via *IMAP* on port *143* (open up this port in your firewall if needed). Set to connect via *STARTTLS*. And also remember to input your full email address as your username, since that's what we also used in our PostgreSQL users database.

There is one more important thing left to do...sending mail.

You may have noticed that when you connect your mailclient and you synchronize your IMAP folders, you only have one folder name *INBOX*. There is no *sent*, *trash* or *draft*.
Dovecot has a very neat way of setting up additional folders and setup whether or not users should be automatically subscribed to them.
For this task, Dovecot has a plugin named  *autocreated*. Open up your *dovecot.conf* file and add the following lines:

	:::bash
	$ protocol imap {
		mail_plugins = $mail_plugins autocreate
	  }
	  plugin {
	    autocreate = Trash
		autocreate2 = Sent
		autosubscribe = Trash
		autosubscribe2 = Sent
	  }

Se first we setup this plugin to be used together with *IMAP*.
Then in the *plugin{}* section we define which folders should be automatically created and which one each user should be automatically subscribed to.
Save this file and restart Dovecot. Now rebuild your *IMAP* folder tree in your mail client...you should see these two new folders appearing!

Now for the actual sending part.

We need to go back to Postfix one more time, because we now need to enable Dovecot *SASL* (remember chapter 1?) and let it use our PostgreSQL database as a lookup for authenticating users before they can send.
Open up your *master.cf* file and add the following overwrites to your submission daemon:

	:::bash
	$  -o smtpd_sasl_type=dovecot
	   -o smtpd_sasl_path=private/auth

This will tell Postfix to not use the default *Cyrus SASL* library, but use Dovecots one, which is simpler to setup and can communicate with PostgreSQL.
Next we need to tell Dovecot to talk *SASL* with Postfix, open up the *dovecot.conf* file and add the following *service auth{}* section:

	:::bash
	$ service auth {
		unix_listener auth_userdb {
			path = /var/spool/postfix/private/auth
			mode = 0660
        	user = postfix
			group = postfix
		}
	 }

Now restart both Postfix and Dovecot.
 
Go back to your mailclient and make sure that you setup *TLS* for the sending server and that is uses *PLAIN* authentication with your full email as your username and also input your password.
Try sending out an email...it should arrive at the other end...and in your sent folder!

There, believe it or not, you are done.

You now have a fully fledged mailserver with Postfix at its heart, PostgreSQL for the bookkeeping, Amavis for virus and spam fighting, Dovecot for the client reception and an "as secure as possible" method woven between them.

Now remember, in these four lengthy chapters we only seen the tip of the iceberg.
Both Postfix and Dovecot are capable of many, many more powerful things...but I suggest that you keep hungry for knowledge and go explore these extra possibilities!

I decided to write a fifth, "unofficial" chapter about setting up some extra functionality to do some more fancy and useful stuff like filtering mail, setting up a vacation responder and an issue named "Time moved backwards". Stay tuned...

And as always, thanks for reading.
