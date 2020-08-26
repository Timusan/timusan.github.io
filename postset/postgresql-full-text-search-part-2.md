<!--
.. title: PostgreSQL: A full text search engine - Part 2
.. slug: postgresql-full-text-search-part-2
.. date: 2014/05/07 13:00:00
.. tags: postgresql, full text search
-->

Welcome to the second installment of our look into full text search within PostgreSQL.

If this is the first time you heard about full text search I highly encourage you to go and read [the first chapter](http://shisaa.be/postset/postgresql-full-text-search-part-1.html "First chapter introducing the full text search capabilities of PostgreSQL.") in this series before continuing. This chapter builds on what we have seen previously.

## A look back

In short, the previous chapter introduced the general concept of full text search, regardless of the software being used. It looked at how the idea of full text search was brought to computer software by breaking it up into roughly three steps: *case removal*, *stop word removal*, normalizing with *synonyms* and *stemming*. 

Next we delved into PostgreSQL's implementation and introduced the *tsvector* and the *tsquery* as two new data types together with a handful of new functions such as *to_tsvector()*, *to_tsquery()* and *plainto_tsquery()*, which all extend PostgreSQL to support full text search. 

We saw how we could feed PostgreSQL a string of text which would then get *parsed* into *tokens* and processed even further into *lexemes* which in turn got *stored* into a *tsvector*. We then queried that *tsvector* using the *tsquery* data type and the *@@* matching operator.

In this chapter, I want to flesh out an important topic we touched on in previously: PostgreSQL's full text search *configurations*. 

## Precaution

Let me be very clear, in *most* cases the configurations shipped with PostgreSQL will suffice and you do not need to touch them at all, in which case this chapter could be considered a waste of time.

*However*, I highly encourage you to read through this chapter and, as always, actually *run the queries* with me. You need to know how the tools you use work under the hood.

To be even more bold, someday you might even need to get your hands dirty and actually *build* your own configuration. Why? Because a customer wanting full text search for their application might have specific requirements, or even deliver you specific dictionaries to use in the parsing stage. Such use cases may arise in very specific areas of conduct where much official, technical lingo is used which is not covered in a general dictionary.

So, put on your favorite pants (or none if you like that better), turn down the lights, pull the computer close to you, open up a terminal window, put on some eery music and let us get started.

## Configuring PostgreSQL full text search

In the last chapter we saw that PostgreSQL uses a couple of tools like a *stop word list* and *dictionaries* to perform its parsing. We also saw that we did not need to tell PostgreSQL about which of these tools to use. It turned out that full text search comes with a set of default configurations for several languages. We also found out that, if no configuration was given, the database assumes that the document or string to be parsed is English and uses a configuration called *'english'*. 

Beware of localized packages of PostgreSQL though. As I noted in the previous chapter, there is a small possibility that the default configuration in your PostgreSQL installation is *not* set to 'english'.
If this is the case with your setup, be sure to include the 'english' configuration if not stated otherwise or *change* it to be 'english'. We will see how to do that in a minute.

Taking the small database we created last time, the syntax to feed a configuration set to PostgreSQL during parsing was the following:

	:::sql
	SELECT to_tsquery('english','elephants & blue');

The string '*english*' represents the *name* of the configuration which we would like to use. As you know by now, this string can be omitted which will make the database use the default configuration. PostgreSQL knows this default because it is set in the general *postgresql.conf* configuration file. In that file you will find a variable called *default_text_search_config* which, in most cases, is set to *pg_catalog.english*. If you wish to have a own, custom configuration to be the default, that is the place to set it.

Before hacking away at your own configuration, it may be of interest to see what PostgreSQL has to offer. To see which shipped configuration files are available to you, use the *describe* command (\d) together with the full text flag (F):

	:::sql
	\dF

This will *describe* the objects in the database that represent full text configurations. You see that by default you have quite a lot of language support. To see a different configuration in action, let us do a quick, fun test. 

First, take the dutch string "Een blauwe olifant springt al dartelend over de kreupele dolfijn.", which is a rough translation of the "The blue elephant jumps over the crippled dolphin." example from the first chapter. If we would feed this to PostgreSQL, using the default (english) configuration:

	:::sql
	SELECT to_tsvector('english', 'Een blauwe olifant springt al dartelend over de kreupele dolfijn');
	
We would get back:

	:::sql
	 'al':5 'blauw':2 'dartelend':6 'de':8 'dolfijn':10 'een':1 'kreupel':9 'olif':3 'springt':4

It attempted to guess some words as you can see from the lexeme 'olif', but, to a dutch reader, this is *not* stemmed correctly. Neither are the stop words removed: 'de' and 'een' are articles which, in dutch, are considered of no value in a text search context. So let us try this again with the built-in *dutch* configuration:

	:::sql
	SELECT to_tsvector('dutch', 'Een blauwe olifant springt al dartelend over de kreupele dolfijn');

And we get:

	:::sql
	 'blauw':2 'dartel':6 'dolfijn':10 'kreupel':9 'olifant':3 'springt':4

Aha! That is much shorter then the previous result, and it is also more correct. As you can see, the words 'de' and 'een' are now removed and the stemming is done correctly on 'dartel', 'olifant' and 'kreupel'.
The target of this series, however, is not to show you the dutch language (for it will make you weep...), but you see the effect a different configuration set can have. 

But what is such a configuration set made of? To answer that, we can simply use the same describe, but ask for more detailed information with the *+* flag:
	
	:::sql
	\dF+
	
This will return a list of *all* the configurations and their details, so let us filter that and look at only the english version:

	:::sql
	\dF+ english
	
The following result will be returned:

	:::bash
	 asciihword      | english_stem
	 asciiword       | english_stem
	 email           | simple
	 file            | simple
	 float           | simple
	 host            | simple
	 hword           | english_stem
	 hword_asciipart | english_stem
	 hword_numpart   | simple
	 hword_part      | english_stem
	 int             | simple
	 numhword        | simple
	 numword         | simple
	 sfloat          | simple
	 uint            | simple
	 url             | simple
	 url_path        | simple
	 version         | simple
	 word            | english_stem
 
 All of these are *token categories* that target the different groups of words that the PostgreSQL full text parser recognizes.
 For each category there are one or more dictionaries defined which will receive the token and try to return a lexeme.
 We also call this overview a configuration map, for it maps a category to one or more dictionaries.
 
 If the parser encounters a URL, for example, it will categorize it as a *url* or *url_path* token and as a result, PostgreSQL will consult the dictionaries *mapped* to this category to try and create a single lexeme containing a URL pointing to the same path. Example:
 
* example.com
* example.com/index.html
* example.com/foo/../index.html

The URLs all result in the same document being served, so it makes sense to only save one variant as a lexeme in the resulting vector.
The same kind of *normalization* is done for file paths, version numbers, host names, units of measure, ... . A lot more then normal, English words.

There are 23 categories in total that the parser can recognize, ones not included here, for example, are *tag* for XML tags, *blank* for whitespace or punctuation, etc.

To see a description of the different token categories supported, use the 'p' flag together with '+' for more information:

	:::sql
	\dFp+

When parsing, the chain of command goes as follows:

* A string is fed to PostgreSQL's full text
* The parser crawls over the string and chops it into tokens of a certain type
* For each token category a list of dictionaries (or a single dictionary) is consulted
* If a dictionary list is used, the dictionaries are (generally) ordered from most precise (narrow) to most generic (wide)
* As soon as a dictionary returns a lexeme (single or in the form of an array), the flow for that token stops
* If no lexeme is proposed (a dictionary returns *NULL*) the token is given to the next dictionary in line or if a stop word list returns a match (returns *empty array*), the token is discarded

## Dictionary templates and dictionaries

In the list of token categories that were supported by the built-in 'english' configuration, you will find that only two *dictionaries* are used: *simple* and *english_stem*, which in turn come from the *simple* and *snowball* dictionary *templates* respectively.

So, what exactly is the difference between a *dictionary template* and a *dictionary*?

A *dictionary template* is the skeleton (hence template) of a dictionary. It defines the actual *C* functions that will do the heavy lifting.
A *dictionary* is an instantiation of that template - providing it with data to work with.

Let me try to clear any confusion on this. 

Take, for example, the *simple* dictionary *template*. It does two things: it first checks a token against a *stop word* list. If it finds a match it returns an *empty array*, which will result in the token being discarded. If no match is found in the stop word list, the process will return the same token, but with *casing removed*.

All the checking and case removing is done by functions, under the hood. The stop word file, however, is something that the *dictionary* (the instantiation) provides.
The instantiation of the *simple* dictionary template, thus the *dictionary* itself, would be defined as follows:

	:::sql
	CREATE TEXT SEARCH DICTIONARY simple (
		template = simple,
		stopwords = english
	);

No need to run this SQL for PostgreSQL already comes shipped with the *simple* dictionary, but I wish to show you how you *could* create it.

First, you will see that we *have* to define the template, thus telling PostgreSQL which set of functions to use.
Next we feed it the data it is expecting, in case of *simple* it only expects a stop word list.

The reason for this separation is a safe guard one. Only a database user with *super user* privileges can write the actual template, because this template will contain functions that, if written incorrectly, could slow down or crash the database. You need someone who knows what they are doing and not your local script kiddy who has normal user access to your part of the database.

Notice that we only give the stopwords attribute the word *english* instead of a full file path.
This is because PostgreSQL has set a few standards in place for all dictionary types we will see in this chapter.

First, in case of a stop word list, the file *must* have the *.stop* extension.

Next, you can provide a full path to the file, anywhere on your system. 
However, if you do not provide a full path, PostgreSQL will search for it inside a directory called *tsearch_data* within PostgreSQL's portion of your system's user *shared* directory.

On a Debian system (using PostgreSQL 9.3) the path to this directory reads: "/usr/share/postgresql/9.3/tsearch_data".

A dictionary like the *simple* dictionary is one that is most of the time put at the beginning of a *dictionary list* to remove all the stop words before other dictionaries are being consulted. However, in all the cases where we see *simple* in the dictionary column of the token type list above, only this dictionary is used, meaning that only stop words are removed and all else is stripped of casing.

## Creating the "simple" dictionary

Say that we wanted to setup our own *simple* dictionary based on the *simple* dictionary template, but feed it our own list of stop words. Before setting up this new dictionary, we would first have to write a stop word file. 

Luckily for us, this is trivial. A stop word file is nothing more then a plain text file with one word on each line. Empty lines and trailing whitespace are ignored. We would then have to save this file with the *.stop* extension. Let us try just that.

Open up your editor and punch in the words "dolphin" and "the", both on their own line. Write the file out as "shisaa_stop.stop", preferably in PostgreSQL's shared directory.

Next we need to setup our dictionary. Connect to the "phrases" database from chapter one and run the following SQL:

	:::sql
	CREATE TEXT SEARCH DICTIONARY shisaa_simple (
		template = simple,
		stopwords = shisaa_stop
	);	

## Setting up a configuration

Now, the dictionary by itself is not very helpful. As we have seen before, we need to map it to token categories before we can actually use it for parsing.
This means that we need to make our own configuration.

Let us setup an empty configuration (not based on an existing one like 'english'):

	:::sql
	CREATE TEXT SEARCH CONFIGURATION shisaa (parser='default');

This statement will create a new configuration for us which is completely empty, it has no mappings. The argument we have to give here can be either *parser* or *copy*. With parser you define which parser to use and it will create an empty configuration. PostgreSQL has only one parser by default which is named...*default*. If you choose *copy* then you will have to provide an *existing* configuration name (like english) from which you would like to make a copy.

To verify that the configuration is empty, run our describe on it:

	:::sql
	\dF+ shisaa
	
And marvel at its emptiness.

Now, let us add the *shisaa_simple* dictionary we created before:

	:::sql
	ALTER TEXT SEARCH CONFIGURATION shisaa
		ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
        WITH shisaa_simple;

As you will see throughout this (and the next) chapter, full text extends not only the data types and functions we have available, but also extends PostgreSQL's SQL syntax with a handful of new statements.
I need to note that all of these statements are *not* SQL standard (for SQL has no full text standard) and thus cannot be easily ported to a different database.
But then again...what is this folly...who would even need a different database!

The new statements introduced here (and in the previous SQL blocks) are:

* *CREATE TEXT SEARCH DICTIONARY*
* *CREATE TEXT SEARCH CONFIGURATION*
* *ALTER TEXT SEARCH CONFIGURATION*

Just remember that these are not part of the SQL standard (something which PostgreSQL holds very dear, in high contrast with many other databases).

Did it work? Well, describe it:

	:::sql
	\dF+ shisaa

And get back:

	:::bash
	 asciihword      | shisaa_simple
	 asciiword       | shisaa_simple
	 hword           | shisaa_simple
	 hword_asciipart | shisaa_simple
	 hword_part      | shisaa_simple
	 word            | shisaa_simple

Perfect!

Here we mapped our fresh dictionary to the token groups "asciihword", "asciiword", "hword", "hword_asciipart", "hword_part", "word", because these will target most of a normal, English sentence.

It is time to try out this new search configuration! Punch in the same on-the-fly SQL as we had in the previous chapter, but this time with *our own* configuration:

	:::sql
	SELECT to_tsvector('shisaa','The big blue elephant jumped over the crippled blue dolphin.');

And we get back:

	:::bash
	'big':2 'blue':3,9 'crippled':8 'elephant':4 'jumped':5 'over':6

Ha! All squeaky flippers unite! The word *dolphin* is *removed*, because we defined it to be a stop word. A world as it should be.

We now have a basic full text configuration with a *simple* dictionary. To have a more real world full text search we will need more then just this dictionary though, we will at least need to take care of stemming.

## Extending the configuration: stemming with the Snowball

Stemming, the process of reducing words to their basic form, is done by a special, dedicated kind of dictionary, the *Snowball* dictionary. 

What?

*Snowball* is a *very proven* string processing language specially designed for stemming purposes and supports a wide range of languages. It originated from the *Porter stemming algorithm* and uses a natural syntax to define stemming rules. 

And luckily for us, PostgreSQL has a *Snowball* dictionary template ready to use. This template has the Snowball stemming rules embedded for a wide variety of languages. Let us create a *dictionary* for our shisaa configuration, shall we?

	:::sql
	CREATE TEXT SEARCH DICTIONARY shisaa_snowball (
		template = snowball,
		language = english
	);	
	
Again, very easy to setup. The snowball dictionary *template* accepts two variables to be setup. The first, mandatory one is the language you wish to support. Without this, the template does not know which of the Snowball stemming rules to take.

The next, optional one is, again, a stop word list. But...why can we feed this dictionary a stop word list? Did we not already do that with the *simple* dictionary?

That is correct, we did setup the *simple* dictionary to remove stop words for us, but we are not required to use the *simple* and the *snowball* dictionary in tandem.
It is perfectly possible to *map* only the *snowball* dictionary for various token categories and ignore all other dictionaries.
If you would not tell the *snowball* dictionary to remove stop words, it could become messy for the Snowball stemmer will try and stem *all* words it finds.

This stop word list can be the exact same list we fed the *simple* dictionary.

Also, because a *snowball* dictionary will try and parse *all* the tokens it is being fed, it is consider to be a *wide* dictionary. Therefor, as we have seen earlier, it is a good idea when chaining dictionaries together to put this dictionary at the end of your chain.

We now have our own version of the *snowball* dictionary and need to extend our configuration and map this dictionary to the desired token categories:

	:::sql
	ALTER TEXT SEARCH CONFIGURATION shisaa
		ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
        WITH shisaa_simple, shisaa_snowball;

Notice that in the *WITH* clause we are now chaining the *simple* and the *snowball* dictionary together. The order is, of course, important.
Describe our configuration once more:

	:::sql
	\dF+ shisaa

And get back:

	:::bash
	asciihword      | shisaa_simple,shisaa_snowball
	asciiword       | shisaa_simple,shisaa_snowball
	hword           | shisaa_simple,shisaa_snowball
	hword_asciipart | shisaa_simple,shisaa_snowball
	hword_part      | shisaa_simple,shisaa_snowball
	word            | shisaa_simple,shisaa_snowball

Perfect, now the *simple* dictionary will be consulted first followed by the *snowball* dictionary.

Note that throughout this chapter I will chain together dictionaries in order. This will *not* always be the most smart or desired order, just an order to demonstrate *how* you can chain dictionaries.

To the test, throw a new query at it:

	:::sql
	SELECT to_tsvector('shisaa','The big blue elephant jumped over the crippled blue dolphin.');
	
And get back:

	:::bash
	'big':2 'blue':3,9 'crippled':8 'elephant':4 'jumped':5 'over':6	

Nice, that is very...oh wait. Something is not correct. I am getting back *exactly* the same result as before. The words "crippled" and "elephant" are not stemmed at all. Why?

Well, the *simple* dictionary, as we defined it earlier, is setup to be a bit greedy. In its current state it will return an unmatched token as a lexeme with casing removed.
It does not return *NULL*. And, as you know by now, *NULL* is needed to give other dictionaries a chance to examine the token.

So, we need to alter the *simple* dictionary's behavior. For this, we can use the *ALTER* syntax provided to us. And as it turns out, the *simple* dictionary *template* can accept one more variable: the *accept* variable. If this is set to false, then it will return *NULL* for every unmatched token. Let us alter that dictionary:

	:::sql
	ALTER TEXT SEARCH DICTIONARY shisaa_simple ( accept = false );

Run the ts_vector query again, and look at the results:

	:::bash
	'big':2 'blue':3,9 'crippl':8 'eleph':4 'jump':5 'over':6

That is what we were looking for, nicely stemmed results!

## Extending the configuration: fun with synonyms

By now we have seen the first and the last dictionary in our control chain, but at least one more important part is missing: synonyms are not removed.

Let us extend our favorite sentence and add a few synonyms to it: "The big blue elephant, joined by its enormous blue mammoth friend, jumped over the crippled blue dolphin while smiling at the orca."

Still perfectly possible.

In the light of (cue dark en deep Batman voice) "science" (end Batman voice), let us first see what we get when we run it through our current configuration:

	:::bash
	'at':20 'big':2 'blue':3,9,16 'by':6 'crippl':15 'eleph':4 'enorm':8 'friend':11 'it':7 'join':5 'jump':12 'mammoth':10 'orca':22 'over':13 'smile':19 'while':18

That is one big result set. Maybe we should cut the blue dolphin a little bit of slack and feed a real stop word list to our *simple* dictionary before continuing by altering our *dictionary*:

	:::sql
	ALTER TEXT SEARCH DICTIONARY shisaa_simple ( stopwords = english );

As you see you can simply use the same *ALTER* syntax as before. The "english" here refers to the shipped "english.stop" stop word list.

Querying again, we will get back a better, short list (including our Dolphin friend):
	
	:::bash
	'big':2 'blue':3,9,16 'crippl':15 'dolphin':17 'eleph':4 'enorm':8 'friend':11 'join':5 'jump':12 'mammoth':10 'orca':22 'smile':19

Now we would like to reduce this result even further by compacting synonyms into one lexeme.

Enter the *synonym* dictionary *template*.

This template requires you to have a so-called "synonym" file; A file containing lists of words with the same meaning. For the sake of learning, let us create our own synonym file. This file has to end with the *.syn* extension.

Open up your editor again and write out a file called "shisaa_syn.syn" with the following contents:

	:::bash
	big enormous
	elephant mammoth
	dolphin orca
	
And let us setup the *dictionary*:

	:::sql
	CREATE TEXT SEARCH DICTIONARY shisaa_synonym (
		template = synonym,
		synonyms = shisaa_syn
	);
	
And add the mapping for it:

	:::sql
	ALTER TEXT SEARCH CONFIGURATION shisaa
		ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
        WITH shisaa_simple, shisaa_synonym, shisaa_snowball;

Okay, time to test our big string again and see the results:

	:::bash
	'blue':3,9,16 'crippl':15 'enorm':8 'enormous':2 'friend':11 'join':5 'jump':12 'mammoth':4,10 'orca':17,22 'smile':19

Very neat. The words "elephant", "big" and "dolphin" are now removed and only their synonyms are kept.
Also notice that both "mammoth" and "orca" have two pointers each, one for every synonym.

But look at the words 'enorm' and 'enormous', why is this happening?

If you look at the pointers, you see that *enormous* points to the second word in the string, being *big*, while *enorm* points to the original *enormous* word.
The reason why this is happening is because our *synonym* dictionary has priority over our *snowball* one. The *synonym* dictionary emits a lexeme as a synonym for *big*, being *enormous*, simply because we told it to do so in our *synonym file*. Now, because it emits a lexeme, the original token, *big*, is not available anymore for the rest of the dictionary chain.

The token *enormous* itself has *no* synonym because we did not define it in our synonym file. It is ignored by the *synonym* dictionary and passed over to the *snowball* dictionary which then stems the token into a lexeme resulting in *enorm*.

If you wish to prevent this from happening, you could add a self pointing line to your synonym list:

	:::bash
	enormous enormous 
	
Now load in the file on disk to pull the changes into PostgreSQL:

	:::sql
	 ALTER TEXT SEARCH DICTIONARY shisaa_synonym (synonyms=shisaa_syn);

And run the query again, the result should now read:

	:::bash
	'blue':3,9,16 'crippl':15 'enormous':2,8 'friend':11 'join':5 'jump':12 'mammoth':4,10 'orca':17,22 'smile':19

Now *enorm* will be removed and both *big* and *enormous* are cast to the same lexeme. 

PostgreSQL does not ship a synonym list, so you will have to compile your own just like we did above but hopefully a little bit more useful

## Extending the configuration: phrasing with a Thesaurus

Next up is the *thesaurus* dictionary, which is quite close to the *synonym* dictionary, with one exception: *phrases*.

A *thesaurus* dictionary is used to recognize phrases and convert them into lexemes with the same meaning. Again, this dictionary relies on a file containing the phrase conversions.
This time, the file has the *.ths* extension. 

Open up your editor and write out a file called "shisaa_thesaurus.ths" with the following contents:

	:::bash
	big blue elephant : PostgreSQL
	crippled blue dolphin : MySQL

Before we can create the dictionary, there is one more required variable we have to set, the *subdictionary* the *thesaurus* dictionary can use.
This subdictionary will be *another* dictionary you have defined before. Most of the time a stemmer is fed to this variable to let the thesaurus stem the input before comparing it with its thesaurus file.

So let us feed it our *snowball* dictionary and set it up:

	:::sql
	CREATE TEXT SEARCH DICTIONARY shisaa_thesaurus (
		TEMPLATE = thesaurus,
		DICTFILE = shisaa_thesaurus,
		DICTIONARY = shisaa_snowball
	);
	
Map it:

	:::sql
	ALTER TEXT SEARCH CONFIGURATION shisaa
		ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
        WITH shisaa_simple, shisaa_thesaurus, shisaa_snowball;

Notice that I took out the *synonym* dictionary. If we chain up to many dictionaries, the results might turn out to be undesirable in our demonstration use case.

Querying will result in the following tsvector:

	:::bash
	'blue':7 'enorm':6 'friend':9 'join':3 'jump':10 'mammoth':8 'mysql':13 'orca':18 'postgresql':2 'smile':15

That is quite awesome, it now recognizes "big blue elephant" as PostgreSQL and "crippled blue dolphin" as MySQL. We have created a *pun-aware* full text search configuration!

As you can see,  both the "MySQL" and "PostgreSQL" lexemes have *one* pointer each, pointing to the first word of the substring that got converted.

## Extending the configuration a last time: morphing with Ispell

Okay, we are almost at the end of the dictionary *templates* that PostgreSQL supports.

This last one is a fun one too. Many Unix and Linux systems come shipped with a spell checker called *Ispell* or with the more modern variant called *HunSpell*.
Besides your average spell checking, these dictionaries are very good at morphological lookups, meaning that they can link all different writing structures of words together.

A synonym or thesaurus dictionary would not catch these, unless explicitly set with a huge amount of lines in the *.syn* or *.ths* files, which is error prone and inelegant. 
The Ispell or Hunspell dictionaries *will* capture these and try to make them into one lexeme.

Before setting up the *dictionary*, we first need to make sure that we have the Ispell or Hunspell dictionary files for the language we wish to support.
Normally you would want to download these files from the official OpenOffice page. These pages, however, seem to be confusing and the correct files very hard to find. I have found [the following page](http://fmg-www.cs.ucla.edu/geoff/ispell-dictionaries.html "OpenOffice Extension page.") of great help to get the files you need for your desired language
.
Download the files for your desired language and place the *.dict* and the *.affix* files into the PostgreSQL shared directory.

For now, let us just take the basic *english* dict and affix files (named both *en_us* and already shipped with PostgreSQL) and feed them to the configuration:

	:::sql
	CREATE TEXT SEARCH DICTIONARY shisaa_ispell (
		template = ispell,
		DictFile = en_us,
		AffFile = en_us,
		StopWords = english
	);

And chain it:

	:::sql
	ALTER TEXT SEARCH CONFIGURATION shisaa
		ALTER MAPPING FOR asciiword, asciihword, hword_asciipart,
                      word, hword, hword_part
        WITH shisaa_simple, shisaa_ispell, shisaa_snowball;

Notice again I took out the *thesaurus* dictionary, not to pile up too many dictionaries at once.

Query it once more, and look at what we get back:

	:::bash
	'big':2 'blue':3,9,16 'cripple':15 'dolphin':17 'elephant':4 'enormous':8 'friend':11 'join':5 'joined':5 'jump':12 'mammoth':10 'orca':22 'smile':19 'smiling':19

Hmm, interesting. Notice that we now got *more* lexemes than before, *smile* and *smiling* for example, and *join* and *joined*. Also, both these cases have the *same* pointer. Why is that?

What is happening here is a feature of the Ispell dictionary called *morphology*, or as we seen above, *morphological lookups*.
One of the reasons why Ispell is such a powerful dictionary is because it can recognize and act upon the *structure* of a word. 

In our case, Ispell recognizes *joined* (or *smiling*) and emits an array of *two* lexemes, the original token converted to a lexeme *and* the stemmed version of the token.

This concludes all the dictionaries that PostgreSQL ships with by default and the ones you will most likely ever need. What is next?

## Debugging

Now that you have a good understanding of how to build your own configuration and setup your own dictionaries, I would like to introduce a few new functions that can come in handy when your configuration would produce seemingly strange results.

### ts_debug()

The first function I want show you is a *very* handy one that is built to test your *whole* full text configuration. It helps you keep your mental condition to just mildly insane, so to speak.

The function *ts_debug()* accepts a configuration and a string of text you wish to test. As a result you will get back a set that contains an overview of how the parser chopped your string into tokens,  which category it picked for each token, which dictionary was consulted and which lexeme(s) where emitted. Oh boy, this is too much fun, let us just try it out! 

Feed our original pun string and let us test the current *shisaa* configuration:

	:::sql
	SELECT ts_debug('shisaa','The big blue elephant jumped over the crippled blue dolphin.');
	
Hmm, that may not be very readable, rather use the wildcard selector and a FROM clause to include column names into our result set (one of the few times you may use this selector without getting smacked):

	:::sql
	SELECT * FROM ts_debug('shisaa','The big blue elephant jumped over the crippled blue dolphin.');	

Which will result in the following, huge set:

	:::bash
      alias   |   description   |  token   |                 dictionaries                  |  dictionary   |  lexemes   
	-----------+-----------------+----------+-----------------------------------------------+---------------+------------
	asciiword | Word, all ASCII | The      | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_simple | {}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | big      | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {big}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | blue     | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {blue}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | elephant | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {elephant}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | jumped   | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {jump}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | over     | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_simple | {}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | the      | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_simple | {}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | crippled | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {cripple}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | blue     | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {blue}
	blank     | Space symbols   |          | {}                                            |               | 
	asciiword | Word, all ASCII | dolphin  | {shisaa_simple,shisaa_ispell,shisaa_snowball} | shisaa_ispell | {dolphin}
	blank     | Space symbols   | .        | {}                                            |               | 

You now have a complete overview of the flow from string to vector of lexemes. Let me go over some interesting facts of this result set.

First, notice how the tokens *the* and *over* got removed by the *simple* dictionary. They where a hit in the stop word list, so the dictionary returned an *empty array*.

Next you see the alias *blank* between each *asciiword*. *Blank* is a category used for spaces or punctuation. A *space* and a *.* (full stop) is considered a token, but is stripped out by the parser itself for it has no value in this context.

And last, see that our *snowball* dictionary was never consulted. This means that, in this string, the *shisaa_ispell* gobbled all the lexemes that *shisaa_simple* threw at it.

### ts_lexize()

The second function is *ts_lexize()*. This little helper lets you test different *parts* of your whole setup. Take the unexpected result of our last dictionary, where we got back multiple lexemes. As it turned out it is normal behavior, but you may want to verify that the result is coming from the dictionary and not from a side effect of how you chained your dictionaries together.

To test our single, *shisaa_ispell* dictionary, we could feed it to this new function, together with *one token* we wish to test:

	:::sql
	SELECT ts_lexize('shisaa_ispell','joined');
	
This will return:

	:::bash
	{joined,join}

Same as we had before, but now we know, for sure, that it is a feature of our Ispell dictionary. 
Notice that I stressed the fact that you can only feed this function *one token*, not a string of text and not multiple tokens.

You can use this function to test all your dictionaries individually, one token at a time.

Phew, that was a lot to take in for we covered a lot of ground here today. You can turn the lights back high and go get some fresh air.
In the next chapter, I will round up this introduction by introducing you to the following, new material:

* Ranking search results
* Highlighting words inside search results
* Creating special full text search indexes
* Setting up update triggers for tsvector records

And as always...thanks for reading!

<!--  LocalWords:  instantiation PostgreSQL
 -->
