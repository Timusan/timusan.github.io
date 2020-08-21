<!--
.. title: PostgreSQL: A full text search engine - Part 1
.. slug: postgresql-full-text-search-part-1
.. date: 2014/04/30 14:00:00
.. tags: postgresql, full text search,
-->

## Preface

PostgreSQL, the database of miracles, the RDBMS of wonders.

People who have read my stuff before know that I am a fan of the blue-ish elephant and I greatly entrust it with my data. 
For reasons why, I invite you to read the "Dolphin ass-whopping" part of the [second chapter](http://shisaa.jp/postset/mailserver-2.html "Second chapter of the mail setup series.") of my mail server setup series.

But what some of you may not know is that PostgreSQL is capable of much more then simply storing and retrieving your data.
Well, that is actually not entirely correct...you are *always* storing and retrieving data.
A more correct way to say it is that PostgreSQL is capable of storing all *kinds* of data and gives you all *kinds* of ways to retrieve it.
It is not limited to *storing* boring stuff like "VARCHAR" or "INT". Neither is it limited to retrieving and *comparing* with boring
operators like "=", "ILIKE" or "~". 

For instance, are you familiar with PostgreSQL's *"tsvector"* data type? Or the *"tsquery"* type? Or what these two represent? No?
Well, diddlydangeroo, then by all means, keep reading, because that is exactly what this series is all about!

In the following three chapters I would like to show you how you can configure PostgreSQL to be a batteries included, blazing fast, competition crunching, full text search engine.

## But, I can already search strings of text with PostgreSQL!

Hmm, that is very correct. But the basic operators you have at your disposal are limited. 

Let me demonstrate.

Imagine we would have a table, called "phraseTable" containing thousands of strings, all saved in a regular, old VARCHAR column named "phrase".
Now we would like to find the string *"An elephant a day keeps the dolphins at bay."*.
We do not fully remember the above string, but we do remember it had the word "elephant" in it.
With regular SQL you could use the "LIKE" operator to try and find a matching substring. The resulting query would look something like this:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase LIKE '%elephant%';

It would work, you render any index on the table mute when using front *and* back wildcards, but it would work.
Now imagine a humble user would like to find the same string but their memory is bad, they thought the word elephant was capitalized, because it may refer to PostgreSQL, of course.
The query would become this:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase LIKE '%Elephant%';

And as a result, you get back zero records.

"But wait!", you shout, "I am a smart ass, there is a solution to this!". And you are correct: the ILIKE operator.
The "I" stands for Insensitive...as in *Case Insensitive*. So you change the query:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase ILIKE '%Elephant%';

And now you will get back a result. Good for you.

A day goes by and the same user comes back and wishes to find this string again. But, his memory still being bad and all, he thought there where multiple elephants keeping the dolphins at bay, because, you know, pun. So the query, you altered yesterday, now reads:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase ILIKE '%Elephants%';
	
And...now the query will return zero results.

"Ha!", you shout in my general direction. "I am a master of Regular Expressions! I shall fix thay query!".

No, you shall *not* fix my query. Never, ever go smart on my derrière by throwing a regular expression in the mix to solve a database lookup problem. It is unreadable, un-scalable and fits only one solution perfectly-ish. And, not to forget, is *slow as hell* for it not only ignores any index you have set, it also asks more of the database engine then a LIKE or ILIKE.

Let me put an end to this and tell you that I am afraid there are no more (scalable) smart ass tricks left to perform and the possibilities to search text with regular, build-in operators are exhausted.

You agree? Yes? Good! So, enter "*full text search*"!

## Full text search?

But before we delve into the details of the PostgreSQL implementation, let us take a step back and first see what exactly a full text search engine is.

Short version: A full text search engine is a system that can retrieve documents, or parts of documents, based on natural language searching.

*Natural language* means the living, breathing language we humans use. And as you know, human language can be complex and above all *ambiguous*.

Consider yourself in the situation where you knew, for sure, that you have read an interesting article about elephants in the latest edition of "Your Favorite Elephant Magazine".
You liked it so much that you want to show it to your best friend, who happens to be an elephant lover too.
The only bummer is, you cannot remember the title, but you do remember it has an interesting sentence in it.

So what do you do? First you quote the sentence in your mind: "The best elephants have a blue skin color.".
Next, you pick up the latest edition and you start *searching*, flipping through the pages, skimming for that sentence.

After a minute or two you shout: "Dumbo!, I have found the article!". You read the sentence out loud: "The best Elephants bear a blue skin tone.".
You are happy with yourself, call up your friend and tell him that you will be over right away to show him that specific article.

One thing you forgot to notice was that the sentence in your head, and the sentence that was actually printed where *different*, but your brain (which is trained in this natural stuff), sees them as the same.
How did that work? Well, your brain used its internal *synonym* list and *thesaurus* to link the different words together, making them the same thing, just written differently:

* "elephants" is the same as "Elephants"
* "have" is the same as "bear"
* "skin color" is the same "skin tone"

Without noticing it, you have just completed a full text search using natural language algorithms, your magazine as the database, your brain as the engine.

## But how does such a natural language lookup work...on an unnatural computer?

What a perfectly timed question, I was just getting to that.

Now that you have a basic understanding of what natural language searching is, how does one port this idea to a stupid, binary ticking tin box?
By dissecting the process we do in our brains, lay it out in several programmable steps and concepts. 
Such a process, run by computers, will never be as good on the *natural* part as our brains are, but it is certainly a lot faster with flipping and skimming through the magazine pages.

Let us look at how a computer, regardless of which program, platform or engine you use, would go about being "natural" when searching for strings of text.

To speed up the search process, a full text search engine will never search through the actual document itself.
That is how we humans would do it, and that is slow and (for our eyes) error prone. Before a document can be searched through with a full text search engine, it has to be parsed into a list of words first.
The parsing is where the magic happens, this is our attempt at programming the natural language process. Once the document is parsed, the parsed state is saved. Depending on your database model, you can save the parsed state together with a reference to the original document for later retrieval.

Note that a document, in this context, is simply a big collection of words contained within a file. The engine does not care, and most of the time does not know, about what kind of file (plain text, LibreOffice document, HTML file, ...) it is handling or what the files structure is. It simply looks at all the readable words inside of the file.

So how does the parsing work? Parsing, in this regard, is all about compressing the text found in a document. Cutting down the word count to the least possible, so later, when a user searches, the engine has to plow through fewer words. This compressing, in most engines, is done in roughly three steps.

### Remove stop words

The first step is the removal of words that do not add any searchable value to the text and are seldom searched for.
These words are known as "stop words", a term first coined by Hans Peter Luhn, a renowned IBM computer scientist who specialized in the retrieval and indexing of information stored in computer systems.

The list of stop words is not limited to simply ones like "and" or "the". There is an extensive list of hundreds and hundreds of words which are generally considered to be of little value in a search context.
A (very) short list of stop words: her, him, the, also, each, was, we, after, been, they, would, up, from, only, you, while, ... .

### Eliminate casing

The following step in the compression process is the elimination of casing - keeping only the lower case versions of a word.
If you would keep a search case sensitive, then "The ElEphAnt" would not match "the elephant", but generally you do want a match to happen.
The user will many times not care (or not know) about casing in a full text search.

### Remove synonyms, employing a thesaurus and perform stemming

The last part in the compacting of our to-be-indexed document is removing words that have the same meaning and perform stemming.
Synonym lookups are used for removing *words* of the same meaning where as thesaurus lookups are used to compact whole *phrases* with similar meaning.

Only one instance of all the synonyms, thesaurus phrases and case eliminations is stored, the surviving word is referred to as a *lexeme*, the smallest, meaningful word.
The lexemes that are stored usually (depending on the engine you use) get an accompanying list of (alpha)numeric values stored alongside. Two types of (alpha)numeric values can be stored in case of PostgreSQL:

* The first type are pure numerical and represent pointer(s) to where the word occurs in the original document.
* The second type is pure alphabetical (actually only capital A,B,C,D) and represent the weight a certain lexeme has. 

Do not worry to much about these two (alpha)numerical values for now, we will get to that later.

Next, let us get practical and start to actually use PostgreSQL to see how all of this works. 

## The tsvector

As PostgreSQL is an *extendable* database engine, two new data types where added to make full text search possible, as you have seen in the beginning.
One of them is called *tsvector*, "ts" for *t*ext *s*earch and "vector", which is analogous with the generic programming data type "vector".
It is the container in which the result of the parsing is eventually stored.

Let me show you an example of such a tsvector, as presented by PostgreSQL on querying.
Imagine a document with the following string of text inside: *"The big blue elephant jumped over the crippled blue dolphin."*.
A perfectly normal sentence, elephants jump over dolphins all the time.

Without bothering about how to do it, if we let PostgreSQL parse this string, we will get the following tsvector stored in our record:

	:::SQL
	'big':2 'blue':3,9 'crippl':8 'dolphin':10 'eleph':4 'jump':5

You will notice a few things about this vector, let me go over them one by one.

* First, you recognize the structure of a vector-ish data type. Hence the name "tsvector".
* Next, the numbers behind the lexemes themselves, like I said before, represent the pointer(s) to that word. Notice the word "blue" in particular, it has two pointers for the two occurrences in the string.
* And last, notice how some lexemes do not even look like English words at all. The lexeme "crippl" or "eleph" do not mean anything, to us humans anyway. These are the surviving lexemes of "cripple" and "elephant". PostgreSQL has stemmed and reduced the words to match all possible variants. The lexeme "crippl", for example, matches "cripple", "crippled", "crippling", "cripples", ... .

Note that the above example is the simplest of full text search parsing results, we did not add any weights nor did we employ a thesaurus (or an advanced dictionary) to get back a more efficient compressing.

Now that we are dwelling inside of PostgreSQL, I can elaborate a bit more about how the parsing works exactly.
As we have seen above, it happens in roughly three steps. But I intentionally neglected to say that with PostgreSQL, there is an intermediate state between the word and the resulting lexeme.

When PostgreSQL parses the string of text it goes over them and first *categorizes* each word into sections like "word", "url", "int", "hword", "asciiword", ... .
Once the words are broken down into categories, we refer to them as *tokens*. This is the intermediate state.
For a token to become a lexeme, PostgreSQL will consult a set of defined *dictionaries* for each category to try and find a match.
If a match is found, the dictionary will propose a lexeme. This lexeme is the one that will finally be put in the vector as the parsed result.

If the dictionaries did not find a match, the word is discarded. The one exception to this are the "stop words", if a word matches a stop word, it will be discarded instead of kept.

Let us now get our hands dirty and setup a quick testing database and rig it up with the phraseTable table we have been using in our journey so far.
But instead of a varchar column, this table will contain a tsvector type for we will unleash to power of Full Text Search!

Note: I am assuming you have at least PostgreSQL 9.1 or higher. This post was written with PostgreSQL 9.3 in mind.

So connect to your PostgreSQL install and create the database:

	:::sql
	CREATE DATABASE phrases;
	
Do not worry to much about the ownership of this database nor the ownership of its tables, you can discard it whole later.
Now, switch over to the database:

	:::sql
	\c phrases

And create the phraseTable table:

	:::sql
	CREATE TABLE phraseTable (phrase tsvector NOT NULL);
	
Okay, simple enough. We now have a tiny database, with a table containing one column of type *tsvector*.

Let us insert a parsed vector into the table.
Again, without employing a thesaurus or any other tools, we only use the built-in, default configuration to parse a string and save it as a vector.

Let us insert the vector, shall we?

	:::sql
	INSERT INTO phraseTable VALUES (to_tsvector('english','The big blue elephant jumped over the crippled blue dolphin.'));

That was easy enough. Most of what you see is simple, regular SQL with one new kid on the block: "*to_tsvector*".
The latter is a *function* that is shipped with PostgreSQL's Full Text Search extension and it does what its name suggests: it takes a string of text and converts it into a *tsvector*.

As a first argument to this function you can optionally input the full text search *configuration* you wish the parser to use. The default is *"english"*, so I could have omitted it from the argument list.
This configuration holds everything that PostgreSQL will employ to do all of the parsing, including a basic dictionary, stop word list, ... .
PostgreSQL has some default settings, which many times are good enough. The 'english' configuration is such an example.

*Note:* As was pointed out by one of my observant readers, depending on how your PostgreSQL is packaged, it could be *localized*. This means that the default 'english' configuration could be changed to reflect the language of the localized package. If this is the case with your install, be sure to *not omit* the optional parameter and keep its value set to 'english' for all the tsvector and tsquery work we will do in this chapter. Otherwise your full text parsing will produce different, unpredictable results which will make this chapter difficult to follow.

In the next chapter we will delve *deep* into creating our own configuration, for now just take it for granted.

If we query the result, with a simple select, it will return our newly created vector:

	:::sql
	SELECT phrase from phraseTable
	
Will return:

	::bash
	'big':2 'blue':3,9 'crippl':8 'dolphin':10 'eleph':4 'jump':5

Now remember that I talked about the second kind of value we could store alongside the numeric pointers, the *weights*? Let us take a deeper look into that now.

First, weights are not mandatory and only give you an extra tool for ranking the results afterwards.
They are nothing more then a label you can put on a lexeme to group it together. With weights you could, for example, reflect the structure the original document had.
You may wish to put a higher weight on lexemes that come from a title element and a lower weight on those from the body text.

PostgreSQL knows four weight labels *A*, *B*, *C*, *D*. The lowest in rank being *D*. In fact, if you do not define any weights to the lexemes inside a tsvector, all of them will implicitly get a *D* assigned.
If all the lexemes in a tsvector carry a *D*, it is omitted from display when printing the tsvector, simply for readability.
The above query result could thus also be written as:

	:::bash
	'big':2D 'blue':3D,9D 'crippl':8D 'dolphin':10D 'eleph':4D 'jump':5D
	
It is *exactly* the same result, but unnecessarily verbose.

I told you, in the very beginning, that a full text engine does not know or care about the structure of a document, it only sees the words.
So how can it then put labels on lexemes based on a document structure that it does not know?

It cannot.

It is your job to provide PostgreSQL with label information when building the tsvector.
Up until now we have been working with simple text strings, which contain no hierarchy. 
If you wish to reflect your original document structure by using weights, you will have to preprocess the document and construct your *to_tsvector* query manually.

Just for demonstration purposes, we could, of course, assign weights to the lexemes inside a simple text string.
The process of weight assignment is trivial. PostgreSQL gives you the appropriately named *setweight* function for this.
This function accepts a tsvector as the first argument and a weight label as the second.

To demonstrate, let me update our record and give all the lexemes in our famous sentence a *A* weight label:

	:::sql
	UPDATE phraseTable SET phrase = setweight(phrase,'A');

If we now query this table, the result will be this:

	:::bash
	'big':2A 'blue':3A,9A 'crippl':8A 'dolphin':10A 'eleph':4A 'jump':5A
	
Simple, right?

One more for fun. What if you wanted to assign different weights to the lexemes?
For this, you have to concatenate several *setweight* functions together.
An example query would look something like this:

	:::sql
	update phraseTable set phrase=
	setweight(to_tsvector('the big blue elephant'), 'A') ||
	setweight(to_tsvector('jumped over the'), 'B') ||
	setweight(to_tsvector('crippled blue dolphin.'), 'C');

The result:

	:::bash
	'big':2A 'blue':3A,7C 'crippl':6C 'dolphin':8C 'eleph':4A 'jump':5B

Not very usefull, but it demonstrates the principle.

If the documents you wish to index have a fixed structure, many times the table that will hold the tsvectors for these documents will reflect that structure with appropriately named columns.
For example, if your document would always have a title, body text and a footer, you could create a table which contains three tsvector type columns, named after each structure type.
When you parse the document and construct the query, you could assign all lexemes that will be stored in the title column with an *A* label, in the body column with a *B* and in the footer column with a *C*.

Okay, that is enough about weights. Simply remember that they give you extra power to influence the search result ranking, if needed.

We now have a table with a decent tsvector inside. The data is in, so to speak. But what can we do with it now?

Well, let us try to retrieve and compare it, shall we!

## The tsquery

You could, of course, simply retrieve the results stored in a *tsvector* by doing a *SELECT* on the column. However, you have no way of filtering out the results using the operators we have seen before (LIKE, ILIKE). Even if you could use them, you would still run into the same kind of problems as before. You would still have a user who will search for a synonym or search for a plural form of the stemmed lexeme actually stored in the vector.

So how do we query it?

Step in *tsquery*. What is it? It is a data type that gives us extra tools to *query* the full *text search* vector.

Pay attention to the fact that we do not call *tsquery* a set of extra *operators* but we call it a *data type*. This is very important to understand.
With *tsquery* we can construct search *predicates*, which can search through a *tsvector* type and can employ specially designed indexes to speed up the process. 

Let me throw a query at you that will try to find the word "elephants" in our favorite string using *tsquery*:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase @@ to_tsquery('english','elephants');
	
Try it out, this will give you back the same result set we had before. Let me explain what just happened.

As you can see, there is again a new function introduced: *to_tsquery* and it is almost identical to its *to_tsvector* counterpart.
The function *to_tsquery* takes one argument, a string containing the *tokens* (not the words, not the lexemes) you wish to search for.
The first argument I give in this example, 'english', is again the full text configuration and is optional.

Let us first look a bit more at this one. Say, for instance, you wish to find two tokens of the sentence inside your database.
Your first instinct would be to query this the following way:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase @@ to_tsquery('elephants blue');
	
Unfortunately, this will throw an error stating there is a syntax error. Why? Because the string your provided as an argument is malformed.
The *to_tsquery* helper function does not accept a simple string of text, it needs a string of tokens *separated by operators*.
The operators at your disposal are the regular *&* (AND), *|* (OR) and *!* (NOT). Note that the *!* operator *needs* the *&* or the *|* operator.

It then goes and creates a true *tsquery* to retrieve the results. Let us try this query again, but with correct syntax this time:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase @@ to_tsquery('elephants & blue');

Perfect! This will return, once more, the same result as before. You could even use parenthesis inside your string argument to enforce grouping if desired.
Like I said before, what this helper function does is translate its input (the tokens in the string) into actual lexemes. After that, it tries to match this result with the lexemes present in the tsvector.

We still have a problem if we would let a user type her or his search string into an interface search box and feed it to *to_tsquery*, for a user does not know about the operators they need to use.
Luckily for us, there is another help function, the *plainto_tsquery* which takes care of exactly that problem: convert an arbitrary string of text into lexemes.

Let me demonstrate:

	:::sql
	SELECT phrase FROM phraseTable WHERE phrase @@ plainto_tsquery('elephants blue');

Notice we did not separate the words with operators, now it is a simple search string. In fact, *plainto_tsquery* converts it to a list of lexemes separated by an *&* (AND) operator.
The only drawback is that this function can only separate the lexemes with an *&* operator.
If you wish to have something other then the *&* operator, you will have to stick to *to_tsquery*.

A word of caution though, the *plainto_tsvector* may seem interesting, but is most of the time not a general solution for building a higher level search interface. When you are building, say, a web application that contains a full text search box, there are a few more steps between the string entered in that box, and the final query that will be preformed. 

Building a web application and safely handling user input that travels to the database is a separate story and *way* beyond the scope of this post, but you will have to build your own parser that sits between the user input and the query.

If you would play dumb and accept the fact that your interface would only allow to enter a string in the search box (no operators, no grouping, ...) then you still need to send over the user input using query parameters *and* you need to make sure that the parameter sent over is a string. This, of course, is not really a parser, this is more basic, sane database handling on the web. 

As tempting (and simple) it might seem to be to build a query like that, it will probably frustrate your end users. The reason why is because as I mentioned before, the *plainto_tsquery* accepts a string, but will chop the string into separate lexemes and put the *&* operator between them. This means that *all* the words entered by the user (or at least their resulting lexemes) must be found in the string.

Many times, this may not be what you want. Users expect to have their search string interpreted as *|* (OR) separated lexemes, or users may want the ability to define these operators themselves on the interface.

So, one way or the other, you will have to write your own parser if you want a non-database user to work with your application. This parser looks at the options your present on your search form and will crawl over the user entered string to interpret certain characters not as words to search but as operators or grouping tokens to build your final query.

But enough about web applications, that is not our focus now. Let us continue.

The next, new item you will see in the last few queries is the *@@* operator. This operator (also referred to as text-search-matching operator) is also specific to a full text search context. It allows you to *match* a *ts_vector* against the results of a *ts_query*. In our queries we matched the result of a *ts_query* against a column, but you could also match against a *ts_vector* on the fly:

	:::sql
	SELECT to_tsvector('The blue elephant.') @@ to_tsquery('blue & the');

A nice little detail about the *@@* operator is that it can also match against a *TEXT* or *VARCHAR* data type, giving you a poor-mans full text capability. Let me demonstrate:

	:::sql
	SELECT 'The blue elephant.' :: VARCHAR @@ to_tsquery('blue & the');

This 'on-the-fly' query will generate a *VARCHAR* string (by using the *::* or *cast* operator) and try to match the tokens *blue* and *the*. The result will be *t*, meaning that a match is found.

Before I continue, it is nice to know that you can always test the result of a *ts_query*, meaning, test the output of what it will use to find lexemes in the *ts_vector*.
To see that output, you simply call it with the helper function, the same way we called the *to_tsvector* a while ago:

	:::sql
	SELECT to_tsquery('elephants & blue');

This will result in:

	:::sql
	 'eleph' & 'blue'

It is also important to note that *to_tsquery* (and *plainto_tsquery*) too uses a configuration of the same kind *to_tsvector* uses, for it too has to do the same parsing to find the lexemes of the string or tokens you feed it. So the first, optional argument to *to_tsquery* is the configuration, this also defaults to "english". This means we could rewrite the query as such:

	:::sql
	SELECT to_tsquery('english','elephants & blue');
	
And we would get back the same results.

Okay, I think this is enough to take in for now. You have got a basic understanding of what full text search means, you know how to construct a vector containing lexemes, pointers and weights. You also know how to build a query data type and perform basic matching to retrieve the text you desire.

In part 2 we will look at how we can dig deeper and setup our own full text search configuration.
We will cover fun stuff like:

* Looking deeper into PostgreSQL's guts
* Defining dictionaries
* Building Stop word lists
* Mapping token categories to our dictionaries
* Defining our own, super awesome full text configuration
* And, of course, more dolphin pun...

In the last part we will break open yet another can of full text search goodness and look at:

* Creating special, full text search indexes
* Ranking search results
* Highlighting word inside search results
* Setting up update triggers for ts_vector records

Hang in there!

And as always...thanks for reading!
