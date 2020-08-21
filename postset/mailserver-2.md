!--
.. title: Installing a fully fledged, ready to use mailserver on CentOS 6 with Postfix, PostgreSQL, Amavis, ClamAV, Spamassassin and Dovecot - Part 2
.. slug: mailserver-2
.. date: 2013/03/21 10:00:00
.. tags: unix, postfix, postgresql
-->

## PostgreSQL - The Solid Bookkeeper

Welcome to the second iteration of our mailserver setup!

I'm happy you lived through [part 1](http://shisaa.jp/postset/mailserver-1.html "First chapter of this mailserver series"), and I promise this part will be shorter.

Before we dive in, let me go over the different challenges we will be tackling shortly.
Remember in the last chapter we setup our conveyor belt to run smoothly and we secured things up where possible?
The focus in this chapter will be about keeping track which users we have and some detailed information about them.

Let me draw a quick outline of this chapter:

* Introducing you to mail setup with a database backend (and briefly beating MySQLs ass)
* Installing and configuring the basics of PostgreSQL
* Setting up the various tables and a database user inside of PostgreSQL
* Setting up the different transports between PostgreSQL and Postfix
* Configuring Postfix to talk with PostgreSQL through the previously setup transports

### Mail setup with a database backend

Okay, what's PostgreSQL, why use it (why not MySQL?) and how does it fit in our mailserver setup?

Good question(s)! First, PostgreSQL is a database server. But you probably figured that out already...
If you are not familiar with PostgreSQL (shame on you!) you could compare it to the o-so-popular MySQL or any other SQL oriented database.

Second, if you have done any professional work on the web (or offline) that requires a secure and solid database solution, you hopefully will not have chosen MySQL to do the job.
Why not? Well, MySQL, apart from being the little adopted, limp child of the closed source gals and guys over at Oracle, MySQL is neither mature, secure, powerful nor SQL standardized.
Let the asswooping commence:

* MySQL has the most bizarre security leaks and defaults (you need to install a package called mysql_secure_installation to...well...secure MySQL).
* Feed MySQL with a million records and it will die a horrible, resource eating death.
* Try rebooting a crashed database (after its horrible death), you will love the way your tables have "crashed and burned".
* To MySQL ACID is like the blood which runs through a genuine Xenomorph Alien's veins.
* Data integrity and reliability...MySQL will give you a looooong blank stare.
* MySQL still thinks Multicore is a kind of pesticide to kill more then one bug at a time.
* Ooh and PostgreSQL was petting MySQLs back, waiting for it to give its first burp! (true story...).

To see it explained in a more detailed way, checkout [this](http://www.youtube.com/watch?v=1PoFIohBSM4 "Youtube video explaining why MySQL is bad.") video.

Anyway...if you are serious about a reliable, scalable and robust database, you'll do PostgreSQL.

Now, for the third part of the question...how does it fit in our factory.

You can store your users and their corresponding data in two basic ways. You can either use Unix its own user system as your mail users or you can use a database backend to store your users information. Both are okay, but only one is scalable.

If you setup a mailserver for your own private use, doing it with Unix its own user system is perfectly okay.
But if you want a mailserver that will host 20 or more users, maybe thousands of users, you'll need something more flexible and something more easy to maintain.
Also, if you would like to write your own configuration (web)front-end for users to make and administer their accounts/mailboxes, you'll also need a database that your application can retrieve and store its data in. If your mail setup is already using a database, you could just as well use the data already available. Life is easy!

In our factory, PostgreSQL would be the bookkeeper who reads all entries made into his huge book by you, the big boss. But this bookkeeper will also work close with his colleague, Dovecot (which we will see in the final chapter) to negotiate usernames, passwords and user settings so the factory customers (the readers of email) can identify themselves at the factory front desk and get or send their mail.

But because this bookkeeper has many sensitive information, his office must be secured...well secured.

### Securing your fresh PostgreSQL installation

First we need to have access control...only a few known people can have access to the bookkeepers office.
Some are simply allowed inside without further check, others need certain protocols and passwords and most are just denied access.
To inspect and setup the correct access policy you will need to crack open the *pg_hba.conf* file.
The 'hba' part stands for host-based authentication. It basically tells which client application can use which method to authorize itself.

In this file you typical define access on a fine grain level. On a per-line basis you can specify the user, an IP-address range the user can come from, the database(s) it can connect to and the method they can use to authenticate themselves. Go ahead and open up the *pg_hba.conf* file. 

On CentOS 6 you can find it under */var/lib/pgsql/9.2/data/pg_hba.conf* (if you have the 9.2 version that is). Once open you can see a bunch of lines already in there, defining the default PostgreSQL access policy.

By default PostgreSQL will only allow Unix domain socket connections or localhost to connect through md5 password authentication.
Lets add a policy for Postfix and Dovecot to access our database server.

Before we edit the file, here are a few basic syntax guidelines to remember:

* Only one host entry per line is allowed and these lines may not wrap
* Each line has multiple sections and each section must be separated by spaces or tabs
* To comment-out a line, use a hash sign in front of it

Dovecot and Postfix connect to PostgreSQL through password authentication, so to enable this, we could simply add the following line:

	:::bash
	$ host	all all	127.0.0.1/32 md5

But, this would enable all databases to be connected to on the tcp/ip protocol.
This could create a possible backdoor, so lets crank it up a bit.

We only want Postfix and Dovecot to be able to connect to our database server through *host*.
So to tighten things up we need to set the database user and database name in the file.
We still have to make that user and database, but for the sake of this post, lets assume it already exists.

The database name will be...well..."mail" and the user we will name "mailreader".
So, if we input that, the line now reads:

	:::bash
	$ host mail mailreader 127.0.0.1/32 md5

But wait, there is one more line to be added for Postfix to be able to connect:

	:::bash
	$ host mail mailreader ::1/128 md5

"::1" is the same as 127.0.0.1, but then for IPv6 addresses.

Be sure to reload or restart PostgreSQL after altering the *pg_hba.conf* file.

A nice-to-know detail: PostgreSQL will read the file from top to bottom. As soon as it finds a matching line to use for the current connection it will use that line and ignore anything under that line.

That's better, now only the database "mail" and the user "mailreader" can connect to PostgreSQL through tcp/ip.

When installing, PostgreSQL made a new user called "postgres" which you can think of as a kind of root user inside of PostgreSQL.
But it has no password setup, so with these settings in your pg_hba.conf, you cannot login yet.
So lets login to PostgreSQL, as root, and setup a password for that user:

	:::bash
	$ sudo -u postgres psql

Then setup the password:

	:::postgresql
	$ ALTER USER postgres PASSWORD 'your-new-password';

Then quit PostgreSQL by using the "\q" command:

	:::bash
	$ \q

Lets go back to the pg_hba.conf file one more time to make sure that local connections (connections through Unix domain sockets) also need a md5 password.
Change the "local" line so it reads md5 at the end:

	:::bash
	$ local  all  all         md5

Restart PostgreSQL and lets start making that database!
Connect to PostgreSQL with the "postgres" user and the accompanying password:

	:::bash
	$ psql -U postgres

First thing to do it to create the database user:

	:::postgresql
	$ CREATE USER mailreader WITH PASSWORD 'foo';

Now we have a user, but this user is a little bit too powerful. It can read and write.
The only one who is allowed to write in our bookkeepers books is the factory boss (that's you).

To tightly secure PostgreSQL for unwanted write access (to the public schemas anyway) we first have to revoke it for everyone:

	:::postgresql
	$ REVOKE CREATE ON SCHEMA public FROM PUBLIC;
	$ REVOKE USAGE ON SCHEMA public FROM PUBLIC;

Then give back access to the *postgres* user itself:

	:::postgresql
	$ GRANT CREATE ON SCHEMA public TO postgres;
	$ GRANT USAGE ON SCHEMA public TO postgres;

From now on, every user that you create does not have write access to the public schema of any database.
I think this is a better default, you now explicitly have to grant write access to a user if you really need it.

Now, lets create the database and the necessary tables for storing our data.
We already know the database is going to be called "mail" so go ahead and create it and grant our fresh user (read) access to it:

	:::postgresql
	$ CREATE DATABASE mail WITH OWNER mailreader;

Now before you continue, make sure you switch to that database, otherwise you will be making tables inside the default *postgres* database:

	:::postgresql
	$ \c mail

Time for the tables inside this database...which tables do we need?
Hmmm, lets think about it, what would our bookkeeper like to register from our customers?
He needs something to identify them with...and some basic information about their mail preferences.

The default style of configuring Postfix mailboxes is to use your Unix user accounts and map them to mailboxes on your harddrive.
Instead of using Unix accounts we will use Postfix's "Virtual Mailbox" mechanism to tie email and destination on your harddrive together.

So lets start by making a table in the bookkeepers book called "users".
In that table we will store the users emailadres, realname, email directory and the users password.

We can also optionally store the Unix *UID* and *GID* of the general owner of the mail directories, but since our mail system will not be based on Unix users and instead is based on database lookups or *virtual* users, we will use only *one* Unix user that will own *all* mail directories and thus the UID and GID will be the same for every user we enter into the database. Because they will be the same we can actually *omit* these two columns. Later we will see how we can tell Postfix and Dovecot about the UID and GID by "hardcoding" them.


	:::postgresql
	$ CREATE TABLE users (
		email TEXT NOT NULL,
		password TEXT NOT NULL,
		realname TEXT,
		maildir TEXT NOT NULL,
		created TIMESTAMP WITH TIME ZONE DEFAULT now(),
		PRIMARY KEY (email)
	);

As you can see we also set some constraints in place, the only thing the user can leave blank is its real name.
I also let PostgreSQL create a primary key on the email column, since we want that one to be unique.
For me, setting a primary key on one or a combination of columns is better them creating an arbitrary integer id column.
If we created an integer id column, we still could have duplicate values in the email column which would break our setup.

The first three columns are quite obvious, but what about the "maildir"?
Well, the "maildir" tells Postfix the location where to read or write new mail from, specific for that user.

Lets set up our first mail user and fill the users table with an entry for that user, shall we?

Before we can make our first entry into the database, we will have to create the main directory where all users will store their mail.
As we said before, their will be *one* Unix user who owns this main directory and all directories under them (the actual mail directories of our virtual users).

That means that the very next thing we have to do is to create that Unix user on your system...or better, to first create a general group that that user will fit into.
So go ahead and create your group called "mail":

	:::bash
	$ groupadd -g 1000 mail

This will create a group with a GID equal to 1000. We will need this number later!
If you get an error stating that the group mail already exists, you can just lookup the GID for that group in the /etc/groups file.

Now you can make the user that owns the mail directories and put it in that group...lets call him "mailreader"

	:::bash
	$ useradd -g mail -u 200 -d /home/mail -s /sbin/nologin mailreader

This will create a user with UID 200, set its home directory to "/home/mail" and make sure that this user cannot log in to your Unix system.
This user is only needed for ownership on the "/home/mailbox" directory, we don't want him/her really logging in and changing stuff...

Before we continue, lets see if the ownership of the mail directory is okay.
Go to that directory and do a ls -la listing to check if "mail" is owned by the correct user and group.

We can now make our first entry into PostgreSQL!
Fire up your PostgreSQL shell and create the user:

	:::postgresql
	$ INSERT INTO users (
		email, 
		password, 
		realname, 
		maildir
	) VALUES (
		'foo@yourdomain.com', 
		md5('ecnrypted_password'), 
		'Foo Lastname', 
		'foo/'
	);

Note that the password should be encrypted. In the statement above I use the -not so strong- md5 without salting.
This is considered bad practice. by me anyway, because this kind of hashing is weak and thus open to dictionary attacks.

I recommend using AES or a SHA variant (SSHA256 or SSHA512) or a BlowfFish (BF) variant. It will ask more of your mailserver CPU, but its much, much more secure.
You can either use an external program to generate your password hash or you could use PostgreSQL's internal functions for that.
If you use PostgreSQL functions to do the hashing job you will have to install the pgcrypto extension in each database you want these functions available.
We will not be using PostgreSQL functions to do the job, simply because it is not all that compatible with the hashing schemes that Dovecot understands.
In the last chapter, when we will learn more about Dovecot, we will use a program called "doveadm" that's shipped with Dovecot to do the job for us.

To generate a SHA512 password with doveadm, issue the following command:

	:::bash
	$ doveadm pw -s SHA512 -p yourpassword
	
Where "yourpassword" should be replaced with your actual, desired password of course.
This will give you back a string which is your SHA512 encrypted password, ready to be put into your PostgreSQL users table record.

But for the sake of knowing PostgreSQL a little bit better, I will explain how you would go about hashing passwords with pgcrypto.
We want a strong bf/5 password, a BlowFish string hashed 5 times. An encryption of this type will take a 1.2 Ghz computer about 33 years to crack passwords only containing a-z.
If you would use plain old md5 it would take that same computer only 1 day.

To make that hash on the fly we can use the crypt() and gen_salt() functions of PostgreSQL, it will do all the hard work for us....don't you just love that PostgreSQL!
But...by default these two function are not available, you will have to install the *contrib* package of the PostgreSQL core team first:

	:::bash
	$ yum install postgresql92-contrib

Then connect to the mail database and load that extension in that database:

	:::postgresql
	$ CREATE EXTENSION pgcrypto;

There, now, only for the mail database, PostgreSQL has loaded up the necessary functions to perform some hashing magic!
Lets try to insert that user again, now with a stronger password hashing.

	:::postgresql
	$ INSERT INTO users (
		email, 
		password,
		realname,
		maildir
	) VALUES (
		'foo@yourdomain.com', 
		crypt('ecnrypted_password',gen_salt('bf',5)), 
		'Foo Lastname', 
		'foo/'
	);

Voila, now we have used PostgreSQL to do the hashing using one of its extensions.

But now, back to a world without pgcrypto...

In the statement we used to insert the new user, Note the "/" behind the users maildir entry. This is necessary for Postfix to function!
You can also notice that I did not input the full directory path eg. "/home/mail/foo/" but just the directory inside of the mail directory.
This is because you told Postfix that the *virtual_mailbox_base* parameter is "/home/mail".

Yes! We have our first user!

Now our bookkeeper knows enough information about the customer named Foo.

But as we have seen before, the bookkeeper does not only hold customer data, it also holds internal factory information.
One of these pieces of information is called a "transport".
Via a transport, Postfix knows where to store emails coming from a certain domain.

Our setup is only on a single domain, so we actually don't need to explicitly define a transport, but, again, for the sake of knowledge, lets make one anyway.
If you have a larger setup where you have one mail server accepting mail from several domains, you need to setup a transport for each single domain.

For us to store this transport information, we will ask our bookkeeper to make a new entry in its large book called "transports":

	:::postgresql
	$ CREATE TABLE transports (
  		domain  TEXT NOT NULL,
		gid INTEGER UNIQUE NOT NULL,
		transport TEXT NOT NULL,
		PRIMARY KEY (domain)
	);

This table is quite self-explanatory, it will contain the domain from which to receive email and the GID that corresponds to the Unix group you just made.
In the transport column we can store which type of transport to use for which domain.
For now we leave this on *virtual:*, this will make a virtual transport for our domain.

"What is a virtual transport?", you ask.

A virtual transport is a default setting within Postfix that tells the system that this domain is the final destination.
So for our factory this entry would be:

	:::postgresql
	$ INSERT INTO transports (
		domain, 
		gid,
		transport
	) VALUES (
		'exmaple.com',
		1000,
		'virtual:'
	);

Finally we need to give information about aliases we want to use.
A user can have multiple email addresses who all link to the same mailbox.
In Postfix this is called "Virtual Maps", but I like to call them aliases...lets create that database table:

	:::postgresql
	$ CREATE TABLE aliases (
  	  alias TEXT NOT NULL,
	  email TEXT NOT NULL,
	  PRIMARY KEY (alias)
    );

And insert some data for demonstration purposes:

	::postgresql
	$ INSERT INTO aliases (
		alias, 
		email
	) VALUES (
		'bar@exmaple.com',
		'foo@example.com'
	);

Good, that's enough information for now.

Before you can continue however, we still need to change the ownership of these tables. You made these tables with the main user "postgres" so the user "mailreader" cannot access them.
To change ownership make sure you are still connected to your database "mail" with the main user "postgres" and issue the following command on the whole database:

	:::postgresql
	$ ALTER DATABASE mail OWNER TO mailreader;

Then alter the user for each table in your database:

	:::postgresql
	$ ALTER TABLE users OWNER TO mailreader;
	$ ALTER TABLE transports OWNER TO mailreader;
	$ ALTER TABLE aliases OWNER TO mailreader;

There, that's better!

But Postfix and PostgreSQL still don't know of each others existence.
PostgreSQL is just sitting there, waiting to be read out, we have to instruct Postfix to get its information not from files but from the database.
This is done by creating mapping files for the different Postfix mechanisms. These files will hold the queries which will return the necessary information to Postfix.
For our setup to work we will have to create 4 different mapping files for transport, uids, gids, mailboxes.

Lets start with our mailboxes. Postfix needs to know which mailboxes exist so that when it receives mail on foo@example.com it knows that it can accept it.
Instead of writing a database connect statement and a real query, the mapping files are in a right side and left side syntax.

Open up a new file called mailboxes.cf and put in your database information:

	:::bash
	$ user=mailreader
	  password=secret
      dbname=mail
      table=users
      select_field=maildir
      where_field=email
      hosts=localhost

This doesn't look to scary, right? We just define the user and password to connect to the "mail" database.
Then we build up our query on the table users and get the "mailbox" field for the current email that Postfix is currently checking for.  
All of these queries are very simple, we just connect to PostgreSQL via the correct privileged user and do a select on the correct table with a where clause.
Now save this file in logical directory, the default is "/etc/postfix/pgsql/mailboxes.cf"

Let continue with the remaining four files. Open up a new file with the name *transport.cf* and build your query:

	:::bash
	$ user=mailreader
      password=secret
      dbname=mail
      table=transports
      select_field=transport
      where_field=domain
      hosts=localhost

Our third file will be called *virtual.cf*:

	:::bash
    $ user=mailreader
      password=secret
      dbname=mail
      table=aliases
      select_field=email
	  where_field=alias
	  hosts=localhost

Now that we have our files ready for Postfix, we still need to tell it to use these files to communicate with PostgreSQL.
For making this happen, open up the good old *main.cf* file from Postfix, go to the bottom and start telling about PostgreSQL!

	:::bash
	$ local_recipient_maps =
	  virtual_uid_maps = static:200
      virtual_gid_maps = static:1000
	  transport_maps = pgsql:/etc/postfix/pgsql/transport.cf
	  virtual_mailbox_base = /home/mail
	  virtual_mailbox_maps = pgsql:/etc/postfix/pgsql/mailboxes.cf
	  virtual_maps = pgsql:/etc/postfix/pgsql/virtual.cf
	  mydestination = $mydomain, $myhostname

Notice the first line "local_recipient_maps =", it has no value at the end. This tells Postfix to turn off this kind of lookup since we only use virtual lookups via PostgreSQL.
Also note that we added the *UID* and *GID* of the Unix user we created above, that owns all mail directories.

Phew, we are almost there, you can now restart Postfix to load in the new changes:

	:::bash
	$ /etc/init.d/postfix restart

Now you should be able to send mail to the addresses you input in your users table.
Go to your "home/mail/USER" directory, it should be empty, but when Postfix delivers its first mail it will make a directory structure in there with three directories: "cur", "new" and "tmp".
Also remember to check your maillog if your encounter any errors!

In the [next chapter](http://shisaa.jp/postset/mailserver-3.html "Third chapter of this mailserver series") we will do some some serious spam and virus fighting!

And as always...thanks for reading!
