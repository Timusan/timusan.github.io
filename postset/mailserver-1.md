<!--
.. title: Installing a fully fledged, ready to use mailserver on CentOS 6 with Postfix, PostgreSQL, Amavis, ClamAV, Spamassassin and Dovecot - Part 1
.. slug: mailserver-1
.. date: 2013/03/08 9:00:00
.. tags: unix, postfix
-->

### Note

1. In this post I assume you have basic Unix knowledge. The intent of every post I write about Unix is to share what I have learned and in turn give you more know-how and make you less afraid of using this operating system.
2. I assume you have a basic CentOS 6 installation (though, with minimal tweaking, it is applicable for most Unix/Linux systems),  that you are root or can use the sudo command and that you are connected to the Internet.
3. I take it for granted that you have a hunger for knowledge and thus don't mind me taking small side steps to explain some things in more detail.
4. And finally I want to thank a good colleague of mine, mister [Sjamaan](http://more-magic.net "Sjamaans homepage"), for shedding some light in a few of the deepest cavities of mail setup.

## Preface

I'm not going to lie to you guys, setting up a mail server can be a pain in the ass.

The road to mail-server-setup is paved with carcasses of system administrators; one hand still clinging to their keyboards and the other reaching out to the light at the end of this gloomy path. 

When stepping over the dead corpses you might notice something odd. Looking closely at the bony fingers you'll find the tips resting on the CTRL key...and the Y key (or CTRL and V key for the none Emacs corpses). Oh my god, that's meaning they where pasting something...right before meeting their maker? 

Yes, they where...they where just copy and pasting the lines of code they found lying around the internet, hoping this way they would reach the land of mail server kingdom.

Helas, they where wrong.

The most important rule is to never just copy and paste, but learn what you are doing.
Open up Duckduckgo.com and search beyond this blog post. Be hungry for knowledge!

The second most important rule to survive this setup is by building it piece by piece. Configure one piece, test it, confirm it is working accordingly and understand what you just did. Only then can you move on to the next bit.

Got it?

Good.

To try and make things more clear, I'll make the analogy to a factory, where our CentOS install is our clean factory floor and we will be building the inside piece by piece. 

And remember, you are the boss of the factory. On your door is a gold plaque with the letters *Postmaster* engraved into it. Sitting in your high office you can instruct and overview everything that's happening while drinking whiskey and smoking cigars. Oh, and you also have an en-suit Jacuzzi, of course.

Are you ready? Lets go!

## Outline

Let me start of by giving you an outline of what we will be doing throughout this mailserver series:

*Chapter 1 - "Postfix, The Conveyor Belt" - About Postfix, SMTP and security*

* Fire up the idea of a mail setup being a factory
* Make sure that you have the latest available packages and if not, learn how to get them
* Install Postfix and take a peek under the hood
* Take a look at our first and most important part of Postfix, the SMTP daemon
* Take a trip to Secure Land where I'll do my best to explain SSL/TLS/STARTTLS
* Crack open Postfix's configuration files and start to setup a basic but secure mailserver, making sure you can send out mail
* Actually sending out your first mail

*Chapter 2 - "PostgreSQL, The Solid Bookkeeper" - About using PostgreSQL as our back-end for keeping track of user settings*
   
* Introducing you to mail setup with a database backend (And briefly beating MySQLs ass)
* Installing and configuring the basics of PostgreSQL
* Setting up the various tables and a database user inside of PostgreSQL
* Setting up the different transports between PostgreSQL and Postfix
* Configuring Postfix to talk with PostgreSQL through the previously setup transports
   
*Chapter 3 - "Milters, The Trusty Workers" - About building a solid protection against all kinds of spam and viruses*

* Install and setup Amavis
* Configure Postfix to let Amavis do some heavy check lifting
* Install and configure ClamAV and make it run with Amavis
* Install and configure Spamassasin and make it run with Amavas

*Chapter 4 - "Dovecot, The Friendly Frontdesk" - About letting your customers read and send their mail*

* Small introduction to Dovecot
* Introduction to IMAP and POP3
* Configure Dovecot to use IMAP
* Setup SSL/TLS inside Dovecot
* Hashing our users passwords with doveadm
* Setting up extra mail folder
* Connecting with an email client

## Chapter 1 - “Postfix, The Conveyor Belt”

The first and most important piece is *Postfix*. This nifty program is our centerpiece; the basis of our mail system. Without it, we'll have a beautiful politician: nice to look at, but does absolutely nothing. Postfix is whats called a *MTA*, which is short for *Mail Transfer Agent*. 

What it basically does is receive the mail, pulls it through various checkpoints and if all is well, drops it off at the correct end point. This endpoint can either be a users local “mailbox” or another MTA on another server.

You can think of it as the main conveyor belt in our factory (hence the word Transport in MTA). It runs through our entire factory, starting and finishing at the same point, making a loop so to speak. This means that the transport is able to receive mail, but also deliver mail through the same “gate” or *port*.

At the sides of the belt are various workers who need to check the email on different criteria like viruses, spam, … . They will pick up the mail from the conveyor belt, check it, and if it is good put it back on the transport. If it is bad they can mark it as so and put it back on the belt or they might discard it all together. It all depends on how you, as the big boss, instruct your workers.

Other workers are stationed after these so called “Milters” and are instructed to read out the recipient name and carry it to their mailbox. Or, if the recipient is not known in this factory, carry it to the boss his mailbox and let him decide what to do with it.

After those we finally have workers who will put new mail onto the conveyor belt to be sent out from this factory to another factory.

But before we can install the conveyor belt and hire all our workers, we'll have to do a little bit of work on the outside of our factory; more specifically our factory name. Otherwise the mailmen cannot find our building, right?

Postfix requires us to have a *Fully Qualified Domain Name* or *FQDN*. Thats our full domain name, including our mail prefix, as our server name. To find out what your current server name is, you can use the command *hostname*:
	
	:::bash
	$ hostname

If it reads some funky name that is not your domain name then the mailmen will never be able find your building and thus Postfix will refuse to start. 

If your domain is *example.org* and you want your mail prefix to be *mail.example.org* then thats what your FQDN should be. 

To change your hostname you have to edit the "/etc/sysconfig/network" file and set the "HOSTNAME=" paramter:

	:::bash
	$ HOSTNAME="mail.example.org"

Be sure to reboot your machine after setting your new hostname.

After you set the server hostname, you will have to tell Postfix about it, but we'll do that later...keep on reading, lets first get ready to install Postfix!

Always try to install the latest, stable packages, *especially on a server thats open to the Internet*. In CentOS, you basically have older, stable packages in your basic repositories. To enable newer packages you have to import a third-party repo. For installing Postfix I'll use *CentOS ALT*.
To find out about different repositories for any Linux distro, not only CentOS, I recommend [pkgs.org](http://pkgs.org "Linux packages website"). It provides you with a clean search engine and a detailed outline of the available packages for your system.

First, import the repository into your installation. To do this, search and install the latest release rpm (in this case Centalt) file to your server:

	:::bash
	$ rpm -Uvh http://centos.alt.ru/repository/centos/6/x86_64/centalt-release-6-1.noarch.rpm

Now, lets start by installing that conveyor belt, shall we?

	:::bash
	$ yum install postfix

There you go, a fresh new conveyor belt, ready to roll...well, almost anyway. First a small peek under Postfix's hood.

Although you installed the Postfix program as one package, it will actually spawn several different daemons controlled by one master daemon. All these daemons are responsible for the different tasks Postfix can perform or that make Postfix interact with other programs. 

The reason for this multi-daemon setup is merely out of security reasons. Because each of these daemons can only perform small tasks and have only limited rights, the impact of a daemon being compromised can be kept very minimal.

Keeping this in mind is crucial for being able to configure Postfix correctly. 

One part of making Postfix tick is to setup the different daemons you would like to use in your system. You don't need more daemons then necessary; this would only consume system resources and bring in possible security threats.

Note: its possible to edit Postfix's configuration without opening any configuration file by using the *postconf* command. I don't endorse using this command because I believe it is essential to see the structure of the configuration files and to know what other (maybe conflicting) values are in there. So in these posts I wont be using postconf but I will edit the configuration files directly instead.

Lets crack open the first configuration file in Postfix: the *master.cf* file.

This file contains the information about all the daemons you want to spawn when running Postfix.

You can consider each daemon the connecting piece between your conveyor belt and the different workers around it. Throughout this post we will be returning to this file when we install new pieces around our conveyor belt.

### SMTP

The first daemon we want to enable is our *SMTP* daemon.

What is SMTP you ask? Its short for Simple Mail Transfer Protocol.

It is a standard in electronic mail transmission that takes care of sending and receiving mail between servers. SMTP creates guidelines for servers so that they can break up and transmit mail messages. It was presented in a [1982 spec](http://tools.ietf.org/html/rfc821 "The 1982 spec about SMTP") as a successor to FTP Mail.

SMTP roughly works as follows: when sending out a mail message with your mail client, all the different parts (sender, recipient, message body, ...) are put together in the form of text strings. These strings are separated by code words that SMTP can interpret so it knows what the "To:" address is, what the "Subject" is and what the "Message Body" contains. SMTP then looks at the "To:" addres(ses) and breaks it up into the user part (before the @) and the domain part (after the @). With this information, SMTP knows to which outside SMTP server to connect to deliver the mail. The receiving SMTP server in turn knows all the different parts and uses the simple commands to see if it knows the user that is in the "To:" part. If the user is known it will deliver the mail into the correct endpoint. When the user retrieves his/her mail with a mail client, that client, in turn, understands in which field to put which text when you are displaying the mail on your screen.

In our factory analogy the SMTP daemon is the workstation sitting between the outside of our factory and the start and endpoint of our conveyor belt. The factory has many gates on which things can be delivered or sent, but the SMTP workstation sits at gate 25.

But hold on...knowing that we want to make a ready-to-use mailserver, hooked up on the internet and thinking of the fact that SMTP uses simple text strings to sent information over, should make you cringe a little bit. Plain text from one server to the other...that can't be good right? Well, in the old days when internet was only used by Unicorns to communicate lovely, fluffy messages to each other, all was fine. But sending pure, unencrypted text messages in this day of age may not be the best idea...

So we need to secure things up...but how?

The short answer is: we could do it through *Transport Layer Security (TLS)* or *Secure Socket Layer (SSL)*.

To fully answer that question, we will have to dive briefly into how current security mechanisms work on the internet.

### Lets go on a small trip to Secure land

You probably have heard of the term *SSL*, which stand for *Secure Socket Layer*. This, *proprietary* protocol was invented by Netscape way back and is intended to secure information and verify ownership. This protocol is mostly used in web browsers to authenticate websites (so you know that you are really visiting the website you want to visit) and encrypt the traveling data (so that nobody can eavesdrop over the wire). You know you are dealing with SSL when the address bar reads https instead of http (in most cases).

TLS does exactly the same thing, but in a slightly different way. Created by the IETF, TLS is actually the successor of SSL.
Or to be more correct, TLS is the *open and standardized* version of SSL.

A common mistake is to think that the main, big difference between SSL and TLS is that TLS can run on the same port where you would also run your insecure traffic. It could "upgrade" that port to accept a secure connection. This thought seems to be one of the biggest myths floating around SSL/TLS. The truth is: both protocols support running on the same port as there insecure variants. Even more so, on a protocol level, the difference between version 3 of SSL (SSLv3) and the first iteration of TLS (TLSv1.0) are so small you could consider both as the thing. So, in theory, you could use both SSL and TLS on the same ports for securing email traffic.

But, if we want to be totally spec complaint (you generally want to be just that) we have to forget about SSL completely.
As the specification states, it is only allowed to use a STARTTLS command followed by a TLS secure connection.

Okay, kinda clear, but how does TLS (or SSL) work underwater?

Hmm, good question, okay, lets make a small sidestep here.

This is where a mechanism called *Public key Cryptography* comes in to play. This mechanism provides each person with 2 keys, a *public* and a *private* one. The public key can be used to encrypt data where as the private key can be used to decrypt that data again and vice versa. The public key is known to the "whole world", the private one is only know to the owner. 

When you send a message to someone, you can encrypt it with that persons public key, because everyone has everyone's public keys (given that you gave everyone your public). The data in that message can in turn only be decrypted with that persons private key. So, if the person really kept that key private, the data is secure. 

And in reverse, if someone encrypts the data with his private key, every one can open it because the public key is known to the world. 

Whats the purpose of doing the latter? 

Well, it proves, without a doubt, that the message was sent by the owner of the private key, no one else could have encrypted that message with that key.
So it works both ways. You can securely sent data, but also securely receive data while verifying it really comes from the person you expect it to come from.

These public and private key pairs can be made and installed by yourself, as we will do further up in this chapter.
But the problem with making them yourself is that it is still difficult to setup trust between you and a person who doesn't know you.
Also, the browser won't recognize your so called *self-signed* certificate and issues a warning to the user saying that its from an untrusted source.

If you setup a personal mail server, as I have, then making a self-signed certificate is perfectly okay and you can safely ignore the browsers initial warning message, because who trusts yourself better then you do, right? 

But if you want to setup a public mailserver, that everyone can visit and use, these keys should be handed out by a third party organization called a "Trusted Registry". Besides the fact that these organizations have too much authority and charge way to much for these certificates, you are still bound to them. They will act as a trusted, third person who checks that the server you want to secure is really yours and, of course, checks your identity. And most (modern) browsers accept these "official" certificates without warning.

*Update*: Recently a new Trusted Registry called *StartSSL* raised itself up and is currently whipping the established mobs ass. They deliver *free*, yes *FREE* certificates. Well, non-wildcard, single domain certificates are free. If you want a wildcard or multi-domain certificate, you also pay just a *fraction* of the price of the more common Mafia certificate issuers.

To acquire such a certificate you register one their [site](http://startssl.com"STARTSSL Website") and you can order a certificate of choice. One cool thing about their way of handling things: you will get a certificate you have to install into your browser to be able to login to your account. Now that is taking security seriously!

So with this new gained knowledge, lets see how the state of affairs is when it comes to securing email with TLS/STARTTLS?

The next two versions of TLS (TLSv1.1 and TLSv1.2) are better prepared for more "recent" attack methods like the BEAST attack and offer enough security upgrades for them to be in favor of TLSv1.0.

But here you are faced with two new issues.

The first one being the client software. Client software wouldn't be client software if they would support these shiny, new protocols.
In many cases...they don't. They still hang around TLSv1.0.
So here you are as a server admin, wanting to give your users the utmost secure protocol available...but the users clients prohibit it.

The second issues is the so called STARTTLS command itself.

The way STARTTLS works is it first establishes an insecure connection. Traveling with that initial connection is a request to the server asking it to start a secure connection.
This command (or request if you will) is, oh yes, a *plain text* command. Knowing it is plain text, a person eavesdropping on your connection (a so called MitM or Man in the Middle) can simply read out that command and alter it. That middle man can delete that request for a secure connection by which the mailserver will setup an unencrypted connection to your client thus our eavesdropper is able to read out your email traffic.

How can I prevent this from happening?

Well...you can't. This really is a client-side issue.

To really prevent a downgrade of the connection, the users mail client has to enforce a secure connection and not accept plain text data traveling back to it.
And to really, really be secure, the user needs to be *trained*. Trained on how the internet works, trained to install a client side certificate on her/his computer, trained not to blindly accept every server-side certificate that comes floating by...but for most email users, this, regretfully, is a bridge to far.

Good, that was a big chunk to chew over in Secure Land, but we made it through!

I advice you, though, to dig deeper into this matter as it is crucial to know how the internet secures itself and how this effects you as a user and a server admin.
For now, lets continue by having a quick look at the different ports available for mail traffic.

There are three commonly known ports for transferring email: The standard SMTP port *25* which can be secured through TLS/SSL using STARTTLS, port *465* which can be secured through SSL, and finally port *587* which, like port 25, can use STARTTLS as a security upgrade request.

The reason for these different ports is a historical one where port 25 being the standard SMTP port already coined in the 1982 spec where SMTP was first introduced. Port 587 being the port that is considered the default port to be used in "modern", secure SMTP transfers, also called the *Submission* port. Port 465 is an "unofficial" port that is not mentioned by the [Internet Mail Consortium spec RFC3207](http://tools.ietf.org/html/rfc3207) and thus considered unorthodox to use.

But if both port 25 and port 587 can be secured in exactly the same way, and are both spec complaint, why not simply use port 25?

Ah, that a good question, another juicy detail to consider is that port 25 is blocked or relayed by most home user ISPs because it is commonly used by many infected, home user PC's to sent out spam. You can find many ranting flame wars about blocking port 25 for this purpose, mainly saying that its like treating long cancer with a cough syrup and that by blocking a valuable port is not tackling the real problem of spam. But, lets not discuss it here.

So in theory it would be best practice to use port 25 (or standardized practice anyway...), but the ISP's, having to much power under there fingertips, make this impossible. Therefor it is agreed upon to use port 587 to setup a secure connection to a mail server.

So our factory opens up gate 587.

Do we also need to bolt down gate 25, because it is used for spam, right?

Ah, euhm...no. We still need this port for communication between the different mailservers themselves.

What we addressed above was the communication between the mail server and the clients computer.
When a user sends an email with his/her client to your server, the submission service (587) will be used.
The client uses an general, not so smart ISP to connect to the internet and thus is effected by port 25 being blocked.

But mailservers, who are connected through the internet via more professional ways, still have access to that port and still use it to communicate between them.

Oh, so we also have to setup encryption on this port 25 too?

I'm afraid not. Welcome to the wonderful world of mail. The traffic between mailserver is mostly *not encrypted*.
And because many servers don't encrypt, your server cannot blindly assume it can send out secure traffic, there is a big chance the receiving server won't understand it.
The only way you can try to get a secure connection is, again, through the STARTTLS command, but as we have seen, that request can be easily removed by an attacker.

So your mails are (mostly) protected when traveling from your desktop to the mailserver, but are just plain text when sent to another mail server.
The only comfort you have here is that its harder to scan traffic between mailservers then it is to scan outgoing traffic from your computer.

### Back to our mailserver

To correctly enable port 587 in our CentOS installation we first need to configure our firewall (assuming you have a firewall installed, which of course is *highly recommended* if you are connect to the internet!). Open up port 587 and be sure to check if port 465 is not open while you're at it, you can also close that one down too. Now this series is not about configuring your firewall and there are many different firewalls out there that you can use, so I will not go into detail on how to change these settings. I currently use IPTables and find it to be quite difficult to configure or read the rendered out configuration file. But that's mainly what you are stuck with, by default, when using Linux.

Ok, we have configured our firewall, lets to the same for our Postfix daemons.
Go ahead, open up the master.cf file:

	:::bash
	$ vi /etc/postfix/master.cf

Uncomment the "submission" line and all the overwrites you find under it.
The overwrites start with a few spaces and the character -o. These lines are used to overwrite configuration variables that can be set in the main.cf file. 
But be careful uncommenting these lines: don't erase the spaces in front, otherwise Postfix won't recognize them as additions to the Submission line.

Eventually the entries should look something like this:

	submission     inet  n       -       n       -       -       smtpd 
		-o syslog_name=postfix/submission
		-o smtpd_tls_wrappermode=no
		-o smtpd_tls_security_level = encrypt
		-o smtpd_sasl_auth_enable=yes 
		-o smtpd_client_restrictions=permit_mynetworks,permit_sasl_authenticated,reject 
		-o milter_macro_daemon_name=ORIGINATING 

Now, what did you actually do?

Well, as you know, the first line you uncommented was to tell postfix to spawn and use the submission service on the SMTP daemon.

The extra overwrites I will explain in a little bit more detail.

	-o syslog_name=postfix/submission

This line merely tells Postfix to write events happening with this daemon in the maillog under the name “postfix/submission”. You can change this to what ever seems fit, but I think the default is already quite decent.

	-o smtpd_tls_wrappermode=no 

This line tells Postfix to not use TLS fallback for older mail clients that don't understand STARTTLS.
	
	-o smtpd_tls_security_level=encrypt

This line enforces TLS to be used. This means that clients who try to make an insecure connection or try to connect with SSL will be blocked by Postfix.

	-o smtpd_sasl_auth_enable=yes

This one enables *SASL* authentication. Another abbreviation...Whats SASL?

SASL stands for Simple Authentication and Security Layer. It is a mechanism that provides a way of secure authentication (username/password) to protocols such as SMTP, POP and IMAP. 

Oh wait...but doesn't STARTTLS use SSL/TLS for security? 

Yes, it uses SSL/TLS, but it also uses SASL, they are complementary.

TLS is used for the “outside” traffic if you will, to encrypt the mails and verify the remote servers identity, as we discussed above. SASL is used to secure the “internal” traffic, the authentication between the mail server protocols and the mail user agent (your mail program). We'll touch more on this in our next chapter.

On to the next line:

	-o smtpd_client_restrictions=permit_mynetworks,permit_sasl_authenticated,reject

By default this important line is empty or filled with a few defaults. Postfix uses this line to see what type of requests it will accept or not.
But, instead of using this line, we will rename it to another restriction list: the "smtpd_recipient_restrictions" list.

Why?

Well, Postfix has around 6 restriction lists that are parsed, one after the other. If one gives the "OK" command, it will continue to the next list.
If you use the *smtp_client_restrictions* list, this will have exactly the same result, but it will happen sooner in the process.
It will happen so soon that the log files will contains not enough information for you to troubleshoot.

If you use the *smtp_recipient_restrictions* list instead, more useful information will be written in the logs (mainly sender and recipient addresses).

**Update :** since Postfix 2.10 it is recommended you use a new setting called *smtp_relay_restrictions* instead and set it up with these defaults:

	-o smtpd_relay_restrictions=permit_mynetworks,permit_sasl_authenticated,defer_unauth_destination

You can specify a comma separated list of values corresponding to different types of requests you want the SMTPS daemon to support.
This line is also our first line of defense against spam. Through a few simplistic mechanisms, Postfix is already able to determine if a message is from a trusted source or not.

We'll keep this line clean and simple, only adding support for local networks, so Postfix can handle mail from clients in the same subnet, and SASL authenticated clients. Everyone else, we reject.

Prior to 2.10, we would use the *smtp_recipient_restrictions* line, at the end of this line it is good practice to set some kind of 'reject' value. This sends a correct reject message back if non of the specified values prior to this reject value where used in the request. This will also help to prevent your mail server of becoming a so called "open-relay server" which spammers can use to send mail through.

In the new *smtp_relay_restrictions*, the final setting *defer_unauth_destination* will take care of this for us. You *don't* need to set the *reject* value anymore.

After reading through this, and you are using a Postfix prior to 2.10, your line should look similar to this:

	-o smtpd_recipient_restrictions=permit_mynetworks,permit_sasl_authenticated,reject

From 2.10 and up it should read:

	-o smtpd_relay_restrictions=permit_mynetworks,permit_sasl_authenticated,defer_unauth_destination

Good, on to the final, funny looking line:

	-o milter_macro_daemon_name=ORIGINATING 

This guy sets the mail filter (milter) macro daemon name to be used.
*Milter* is an API that enables programs outside Postfix (our "Milter" workers at the sides of the conveyor belt) to tap into the transport and inspect SMTP events and mail content for viruses and spam.
We will touch on this in one of the future chapters. 

You have setup your SMTP daemon. You can now save that file.

Remember in the beginning of the chapter we where talking about setting up a FQDN? Now its time to tell Postfix about it.
We will do that by setting the myhostname parameter in the main.cf file. By setting this parameter Postfix has enough information to derive your base hostname, mydomain parameter in the same main.cf file. So you dont need to set this separately.

To set this, open up the main.cf file
	
	:::bash
	$ vi /etc/postfix/main.cf
	
Then find the *myhostname* parameter. You will see a small explanation about this parameter and one or two myhostname sections in comment. It is good practice to not alter these lines by uncommenting them, but to just add your own, new line under it. This way you keep the example intact. If your FQDN was mail.example.org then insert that value:

	:::bash
	$ myhostname = mail.example.org

Don't exit the file just yet, because here are two more parameters that are best to be set: *myorigin* and *mydestination*.
The myorigin parameter is used by Postfix to set a correct domain when users sent email that contains no domain in the envelope or header adres. You could set *myorigin* to equal *myhostname*:

	:::bash
	$ myorigin = $myhostname

But then emailadresses who had an empty header or envelope domain will have an email adres set like so: info@mail.example.org...that does not look pretty.
You probably want it to be info@example.org. To achieve this set the *myorigin* to equal *mydomain*:
	
	:::bash
	$ myorigin = $mydomain

Et voila, now its a pretty mail adres. Remember, you did not need to set the *mydomain* parameter because Postfix automatically derives it from *myhostname*.

The final parameter for now is *mydestination*.

This one tells Postfix from which local domains to receive mail from. In most mail setups that use one computer to handle all daemons, the default setting is perfect. The default settings tells Postfix to only receive mail from the *myhostname* and local.mydomain. Check this in your main.cf file.

That's it for now!

For Postfix to pick up all these changes you have to reload or restart the daemon.
I'm always in favor of totally restarting the daemon, but thats just my paranoia. Issue the following command to restart the daemon:

	:::bash
	$ /etc/init.d/postfix restart

There, you have a basic Postfix working extended by a secure version of SMTP.

Try it, send email to a mailbox outside your system. 
Use the mail command to send mail like so:

	:::bash
	$ mail foobar@domain.tld

If you dont have the mail command, which is the case if you had a bare minimums install of CentOS, then install it like so:
	
	:::bash
	$ yum install mailx

Then enter a subject and press enter. Finally enter some body text and press CTRL-D to send the mail and exit the program. The mail should now arrive in your external inbox.

Now, chances are that you already can receive mail because you did not block port 25.
Try replying to the email you just sent, and punch in the mail command after a few seconds.
There should be a new mail waiting for you!

Now, before we go, lets create our certificate and tell Postfix about it, we will need it in the next chapters.
For our TLS to work we need at least 2 files, a key file and a certificate file. These files should both be in PEM format.
Thank to the OpenSSL library in Unix we can simply make the certificate ourselves, the so called "self-signed" certificates as we have discussed above.
We will be storing these certificates in a "certs" directory in "etc/postfix". So first, go ahead and create the "certs" directory:

	:::bash
	$ mkdir certs

Then issue OpenSSL to create the key and certificate file that will have a lifetime of about 10 years:
	
	:::bash
	$ openssl req -new -x509 -days 3650 -nodes -out /etc/postfix/certs/cert.pem -keyout /etc/postfix/certs/key.pem

Next we have to tell Postfix where these files are located. This can be added in your "main.cf" as such:
	
	:::bash
	$ smtpd_tls_key_file = /etc/postfix/certs/key.pem
	  smtpd_tls_cert_file = /etc/postfix/certs/cert.pem

Remember to restart Postfix:

	:::bash
	$ /etc/init.d/postfix restart

Congratulations! The basics of our conveyor belt are running smoothly!

Take a rest, drink some tea, you deserved it!
In chapter 2, which will be online soon, we'll look at how we can fit the PostgreSQL database into our mailsetup for smooth user and mailbox management.

PS: No mail arrived on your external mailbox? Then our maillog is our friend!

Postfix logs all events in a file called maillog. You can find this file together with all other logs in /var/log.
Go to the end of the file to find out the most recent error messages and see what could have went wrong. Common pitfalls are a firewall still blocking port 587 or syntax errors in your main.cf or master.cf file.

And as always...thanks for reading!
