<!--
.. title: PostgreSQL: A full text search engine - Part 3
.. slug: postgresql-full-text-search-part-3
.. date: 2014/05/14 9:00:00
.. tags: postgresql, full text search
-->

And so we arrive at the last part of the series.

If you have not done so, please read [part one](http://shisaa.be/postset/postgresql-full-text-search-part-1.html "First chapter introducing the full text search capabilities of PostgreSQL.") and [part two](http://shisaa.be/postset/postgresql-full-text-search-part-2.html "Second chapter introducing the full text search capabilities of PostgreSQL.") before embarking.

Today we will close up the introduction into PostgreSQL's full text capabilities by showing you a few aspects I have intentionally neglected in the previous parts. The most important ones being ranking and indexing.

So let us take off right away!

## Ranking

Up until now you have seen what full text is, how to use it and how to do a full custom setup. What you have not yet seen is how to *rank* search results based on their relevance to the search query - a feature that most search engines offer and one that most users expect.

However, there is a problem when it comes to ranking, it something that is somewhat *undefined*. It is a gray area left wide open to interpretation. It is almost...personal.

In its core, ranking within full text means giving a document a place based on how many times certain words occur in a document, or how close these words are relevant to each other. So let us start there.

### Normal ranking

The first case, ranking based on how many times certain words occur, has a accompanying function ready to be used: *ts_rank()*. It accepts a mandatory *tsvector* and a *tsquery* as its arguments and returns a float which represents how high the given document ranks. The function also accepts a *weights array* and *normalization integer*, but that is for later down the road.

Let us test out the basic functionality:

	:::sql
	SELECT ts_rank(to_tsvector('Elephants and dolphins do not live in the same habitat.'),
		           to_tsquery('elephants'));

This is an regular old 'on the fly' query where we feed a string which we convert to a tsvector and a *token* which is converted to a tsquery. The ranking result of this is:

	:::bash
	0.0607927

This does not say much, does it? Okay, let us throw a few more tokens in the mix:

	:::sql
	SELECT ts_rank(to_tsvector('Elephants and dolphins do not live in the same habitat.'),
		           to_tsquery('elephants & dolphins'));

Now we want to query the two tokens *elephants* and *dolphins*. We chain them together in an AND (*&*) formation. The ranking:

	:::bash
	0.0985009
	
Hmm, getting higher, good. More tokens please:

	:::sql
	SELECT ts_rank(to_tsvector('Elephants and dolphins do not live in the same habitat.'),
		           to_tsquery('elephants & dolphins & habitat & living'));

Results in:

	:::bash
	0.414037
	
Oooh, that is quite nice. Notice the word *living*, the *tsquery* automatically stems it to match *live*, but that is, of course, all basic knowledge by now.

The idea here is simple, the more tokens match the string, the higher the ranking will be. You can use this float to later on sort your results.

### Normal ranking with weights

Okay, let us spice things up a little bit, let us look at the *weights array* that could be set as an optional parameter.

Do you remember the weights we saw in chapter one? A quick rundown: You can optionally give weights to lexemes in a tsvector to group them together. This is, most of the time, used to reflect the original document structure within a tsvector. We also saw that, actually, all lexemes contain a standard weight of '*D*' unless specified otherwise.

Weights, when ranking, define importance of words. The *ts_rank()* function will automatically take these weights into account and use a *weights array* to influence the ranking float. Remember that there are only four possible weights: *A*, *B*, *C* and *D*. 

The weights array has a default value of:

	:::bash
	{0.1, 0.2, 0.4, 1.0}

These values correspond to the weight letters you can assign. Note that these are in reverse order, the array represents: {D,C,B,A}.

Let us test that out. We take the same query as before, but now using the *setweight()* function, we will apply a weight of *C* to all lexemes:

	:::sql
	SELECT ts_rank(setweight(to_tsvector('Elephants and dolphins do not live in the same habitat.'),'C'),
	               to_tsquery('elephants & dolphins & habitat & live'));

The result:

	:::bash
	0.674972
	
Wow, that is a lot higher then our last ranking (which had an implicit, default weight of *D*).
The reason for this is that the floats in the weights array *influence* the ranking calculation.
Just for fun, you can override the default weights array, simply by passing it in as a first argument.
Let us put the weights all equal to the default of *D* being *0.1*:

	:::sql
	SELECT ts_rank(array[0.1,0.1,0.1,0.1],
		           setweight(to_tsvector('Elephants and dolphins do not live in the same habitat.'),'C'),
				   to_tsquery('elephants & dolphins & habitat & live'));	

And we get back:

	:::bash
	0.414037
	
You can see that this is now back to the value we had before we assigned weights, or in other words, when the implicit weight was *D*. You can thus influence what kind of an effect a certain weight has in you ranking. You can even reverse the lot and make a D have a more positive influence then an A, just to mess with peoples heads.

### Normal ranking, the fair way

Not that what we have seen up until now was unfair, but is does not take into account the length of the documents searched through

Document length is also an important factor when judging the relevance. A short document which matches on four or five tokens has a different relevance than a three times as long document which matches on the same amount of tokens. The shorter one is probably more relevant then the longer one.

The same ranking function *ts_rank()* has an extra, final optional parameter that you can pass in called the *normalization integer*. This integer can have a combination of seven different values, they can be a single integer or mixed with a pipe (|) to pass in multiple values.

The default value is *0* - meaning that it will ignore document length all together, giving us the more "unfair" behavior. The next values you can give are *1*, *2*, *4*, *8*, *16* and *32* which stand for the following manipulations of the ranking float:

* 1: It will divide the ranking float by the sum of 1 and the logarithmic number of the document length. The latter number is the ratio this document has compared to the other documents you wish to compare.
* 2: Simply divides the ranking float by the length of the document.
* 4: Divides the ranking float by the harmonic mean (the fair average) between matched tokens. This one is only uses by the other ranking function *ts_rank_cd*.
* 8: Divides the ranking float by the number of *unique* words that are found in the document. 
* 16: Divides the ranking float by the sum of 1 and the logarithmic number of the number of *unique* words found in the document.
* 32: Simply divides the ranking float by *itself* and adds one to that.

These are a lot of values and some of them are quite confusing. But all of these have only one purpose: to make ranking more "fair", based on various use cases.

Take, for example, *1* and *2*. These calculate document length by taking into account the amount of *words* present in the document.
The *words* here reference the amount of *pointers* that are present in the tsvector.

To illustrate, we will convert the sentence "These token are repeating on purpose. Bad tokens!" into a tsvector, resulting in:

	:::bash
	'bad':7 'purpos':6 'repeat':4 'token':2,8

The *length* of this document is *5*, because we have *five pointers* in total.

If you now look at the integers *8* and *16*, they take the *uniqueness* to calculate document length.
What that means is they do not count the pointers, but the actual *lexemes*.
In the above tsvector and thus would result in a length of *4*.

All of these manipulations are just different ways of counting document length.
The ones summed up in the above integer list are mere educated guesses at what most people desire when ranking with a full text engine.
As I said in the beginning, it is a gray area, left open for interpretation.

Let us try to see the different effects that such an integer can have.

First we need to create a few documents (tsvectors) inside our famous phraseTable (from the previous chapters) that we will use throughout this chapter.
Connect to your phrase database, add a "title" column, truncate whatever we have stored there and insert a few variable length documents based on Edgar Allan Poe's "The Raven".
I have prepared the whole syntax below, this time you may copy-and-paste:

	:::sql
	ALTER TABLE phraseTable ADD COLUMN title VARCHAR;
	TRUNCATE phraseTable;
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more."'), 'Tiny Allan');
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore.'), 'Small Allan');
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore. And the silken sad uncertain rustling of each purple curtain Thrilled me - filled me with fantastic terrors never felt before; So that now, to still the beating of my heart, I stood repeating, "Tis some visitor entreating entrance at my chamber door - Some late visitor entreating entrance at my chamber door - This it is, and nothing more."'), 'Medium Allan');
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore. And the silken sad uncertain rustling of each purple curtain Thrilled me - filled me with fantastic terrors never felt before; So that now, to still the beating of my heart, I stood repeating, "Tis some visitor entreating entrance at my chamber door - Some late visitor entreating entrance at my chamber door - This it is, and nothing more."  Presently my soul grew stronger; hesitating then no longer, "Sir," said I, "or Madam, truly your forgiveness I implore; But the fact is I was napping, and so gently you came rapping, And so faintly you came tapping, tapping at my chamber door, That I scarce was sure I heard you"- here I opened wide the door; - Darkness there, and nothing more.'), 'Big Allan');

Nothing better then some good old Edgar to demonstrate a full text search ranking. Here we have four different lengths of the same verse making for four documents of different lengths stored in our tsvector column. Now we would like to search through these documents and find the keywords *'door'* and *'gently'*, ranking them as we go.

For later reference, let us first count how many times our keywords occur in the sentence:

* Tiny Allan: "door" 2, "gently" 1
* Small Allan: "door" 2, "gently" 1
* Medium Allan: "door" 4, "gently" 1
* Big Allan: "door" 6, "gently" 2

First, let us simply rank the result with the default normalization of *0*:

	:::sql
	SELECT title, ts_rank(phrase, keywords) AS rank
		FROM phraseTable, to_tsquery('door & gently') keywords
		WHERE keywords @@ phrase;

Before we go over the results, a little bit about this query for people who are not so familiar with this SQL syntax.
We do a simple *SELECT* from a data set using *FROM* filtering it with a *WHERE* clause.
Going over it line by line:

	:::sql
	SELECT title, ts_rank(phrase, keywords) AS rank
	
We *SELECT* on the *title* column we just made and on a "on-the-fly" column we create for the result set named *rank* which contains the result of the *ts_rank()* function.

	:::sql
	FROM phraseTable, to_tsquery('door & gently') keywords
	
In the *FROM* clause you can put a series of statements that will deliver the data for the query. In this case we take our normal database *table* and the result of the *to_tsquery()* function which we name *keywords* so we can use it throughout the query itself.

	:::sql
	WHERE keywords @@ phrase;

Here we filter the result set using the *WHERE* clause and the *matching* operator (@@). The @@ is a Boolean operator, meaning it will simply return *true* or *false*.
So in this case, we check if the result of the *to_tsquery()* function (named keywords and which will return lexemes) *match* the results of the phrase *column* from our table (which contains *tsvectors* and thus lexemes). We want to rank only those phrases that actually contain our keywords.

Now, back to our ranking. The result of this query will be:

	:::bash
	   title     |   rank    
	--------------+-----------
	Tiny Allan   | 0.0906565
	Small Allan  | 0.0906565
	Medium Allan | 0.0906565
	Big Allan    |   0.10109

Let us order the results first, so the most relevant document is always on top:

	:::sql
	SELECT title, ts_rank(phrase, keywords) AS rank
		FROM phraseTable, to_tsquery('door & gently') keywords
		WHERE keywords @@ phrase
		ORDER BY rank DESC;	
		
Result:

	:::bash
	   title     |   rank    
	--------------+-----------
	Big Allan    |   0.10109
	Tiny Allan   | 0.0906565
	Small Allan  | 0.0906565
	Medium Allan | 0.0906565

"Big Allen" is on top, for it has more occurrences of the keywords "door" and "gently".
But to be fair, in ratio "Tiny Allan" has almost the same amount of occurrences of both keywords. Three times less, but it also is three times as small.

So let us take document length (based on *word count*) into account, setting our *normalization* to *1*:

	:::sql
	SELECT title, ts_rank(phrase, keywords, 1) AS rank
		FROM phraseTable, to_tsquery('door & gently') keywords
		WHERE keywords @@ phrase
		ORDER BY rank DESC;	

You will get:

	:::bash
	   title     |   rank    
	--------------+-----------
	Tiny Allan   | 0.0181313
	Small Allan  | 0.0151094
	Big Allan    | 0.0145124
	Medium Allan |  0.013831

This could be seen as a more fair ranking, "Tiny Allan" is now on top because, considering its *ratio*, it is the most relevant. "Medium Allan" falls all the way down because it is almost as big as "Big Allan", but contains lesser occurrences of the keywords. In total five keywords in contrast to "Big Allan" who has eight but is only slightly bigger.

Let us do the same, but count the document length based on the *unique* occurrences using integer *8*:

	:::sql
	SELECT title, ts_rank(phrase, keywords, 8) AS rank
		FROM phraseTable, to_tsquery('door & gently') keywords
		WHERE keywords @@ phrase
		ORDER BY rank DESC;	
		
The result:

	:::bash
	Tiny Allan   | 0.00335765
	Small Allan  | 0.00161887
	Medium Allan | 0.00119285
	Big Allan    | 0.00105303
	
That is a very different result, but quite what you should expect.

We are searching for only *two* tokens here, and considering the fact that uniqueness is now adhered, all the extra occurrences of these words are ignored.
This means that for the ranking algorithm, all the documents we searched through (which all have at least one occurrence of each token) get normalized to only *2* matching tokens.
And in that case, the shortest document wins hands down, for it is seen as most relevant. As you can see in the result set, the documents are neatly ordered from tiny to big.

### Ranking with density

Up until now we have seen the "normal" ranking function *ts_rank()*, which is the one you will probably use the most.

There is, however, one more function at our direct disposal called *ts_rank_cd()*. The *cd* stands for *Cover Density* and is simply yet another way of considering relevance.
This function has exactly the same required and optional arguments, it simply counts relevancy differently.
Very important for this function to work properly is that you do not let it operate on a *stripped* tsvector.

A stripped tsvector is one that has been undone of its pointer information. If you know that you do not need this pointer information - you just need to match tsqueries against the lexemes in you tsvector - you can strip these pointers and thus make for smaller footprints in your database.

In case of our cover density ranker, it needs this positional pointer information to see how *close* the search tokens are to each other.
It makes sense that this ranking function only works on multiple tokens, on single tokens it is kind of pointless.

In a way, this ranking function looks for *phrases* rather then single tokens; the closer lexemes are together, the more positive influence they will have on the resulting ranking float.

In our "Raven" examples this might be a little bit hard to see, so let me demonstrate this with a couple of new, on-the-fly queries.

We wish to search for the tokens *'token'* and *'count'*.

First, a sentence in which the searched for tokens are wide apart: "These tokens are very wide apart and therefor do not count as much.":

	:::sql
	SELECT ts_rank(phrase, keywords, 8) AS rank
		FROM to_tsvector('These tokens are very wide apart and do not count as much.') phrase, to_tsquery('token & count') keywords
		WHERE keywords @@ phrase
		ORDER BY rank DESC;	

Will have this tsvector:

	:::bash
	'apart':6 'count':10 'much':12 'token':2 'wide':5

And this result:

	:::bash
	 0.008624
	 
Let us put these tokens closer together now: "These tokens count for much now that they are not so wide apart!":

	:::sql
	SELECT ts_rank(phrase, keywords, 8) AS rank
		FROM to_tsvector('These tokens count for much now that they are not so wide apart!') phrase, to_tsquery('token & count') keywords
		WHERE keywords @@ phrase
		ORDER BY rank DESC;

The vector:

	:::bash
	'apart':13 'count':3 'much':5 'token':2 'wide':12

The result:

	:::bash
	0.0198206
	
You can see that both the vectors have *exactly* the same lexemes, but different pointer information.
In the second vector, the tokens we searched for are next to each other, which results in a ranking float that is more then double of the first result.

This demonstrates the working of this function. The same optional manipulations can be passed in (weights and normalization) and they will have roughly the same effect.

Pick the ranking function that is best fit for your use case.

It needs to be said that the two ranking functions we have seen so far are officially called *example functions* by the PostgreSQL community.
They are functions devised to be fitting for most purposes, but also to demonstrate how you could write your own.

If you have very specific use cases it is advised to write you own ranking functions to fit your exact needs.
But this is considered beyond the scope of this series (and maybe also beyond the scope of your needs).

## Highlight your results!

The next interesting thing we can do with the results of our full text is to highlight the relevant words.

As is the case with many search engines, users want to skim over an excerpt of each result to see if it is what they are searching for.
For this PostgreSQL delivers us yet another function: *ts_headline()*.

To demonstrate its use, we first have to make our small database a little bit bigger by inserting the original text of the "Raven" next to our tsvectors.
So, again, copy and past this new set of queries (yes you may...):

	:::sql
	TRUNCATE phraseTable;
	ALTER TABLE phraseTable ADD COLUMN article TEXT;
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more."'), 'Tiny Allan', 'Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more."');
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore.'), 'Small Allan', 'Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore.');
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore. And the silken sad uncertain rustling of each purple curtain Thrilled me - filled me with fantastic terrors never felt before; So that now, to still the beating of my heart, I stood repeating, "Tis some visitor entreating entrance at my chamber door - Some late visitor entreating entrance at my chamber door - This it is, and nothing more."'), 'Medium Allan', 'Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore. And the silken sad uncertain rustling of each purple curtain Thrilled me - filled me with fantastic terrors never felt before; So that now, to still the beating of my heart, I stood repeating, "Tis some visitor entreating entrance at my chamber door - Some late visitor entreating entrance at my chamber door - This it is, and nothing more');
	INSERT into phraseTable VALUES (to_tsvector('Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore. And the silken sad uncertain rustling of each purple curtain Thrilled me - filled me with fantastic terrors never felt before; So that now, to still the beating of my heart, I stood repeating, "Tis some visitor entreating entrance at my chamber door - Some late visitor entreating entrance at my chamber door - This it is, and nothing more."  Presently my soul grew stronger; hesitating then no longer, "Sir," said I, "or Madam, truly your forgiveness I implore; But the fact is I was napping, and so gently you came rapping, And so faintly you came tapping, tapping at my chamber door, That I scarce was sure I heard you"- here I opened wide the door; - Darkness there, and nothing more.'), 'Big Allan', 'Once upon a midnight dreary, while I pondered, weak and weary, Over many a quaint and curious volume of forgotten lore, While I nodded, nearly napping, suddenly there came a tapping, As of some one gently rapping, rapping at my chamber door. "Tis some visitor," I muttered, "tapping at my chamber door - Only this, and nothing more." Ah, distinctly I remember it was in the bleak December, And each separate dying ember wrought its ghost upon the floor. Eagerly I wished the morrow - vainly I had sought to borrow From my books surcease of sorrow - sorrow for the lost Lenore - For the rare and radiant maiden whom the angels name Lenore - Nameless here for evermore. And the silken sad uncertain rustling of each purple curtain Thrilled me - filled me with fantastic terrors never felt before; So that now, to still the beating of my heart, I stood repeating, "Tis some visitor entreating entrance at my chamber door - Some late visitor entreating entrance at my chamber door - This it is, and nothing more."  Presently my soul grew stronger; hesitating then no longer, "Sir," said I, "or Madam, truly your forgiveness I implore; But the fact is I was napping, and so gently you came rapping, And so faintly you came tapping, tapping at my chamber door, That I scarce was sure I heard you"- here I opened wide the door; - Darkness there, and nothing more.');

Good, we now have the same data, but this time we stored the text of the original document alongside the "vectorized" version.

The reason for this being that this *ts_headline()* function searches in the original documents (being our *article* column) rather that in your ts_vector column.
Two arguments are mandatory: the original article and the ts_query. The optional arguments are the full text configuration you wish to use and a string of additional, comma separated options.

But first, let us take a look at its most basic usage:

	:::sql
	SELECT title, ts_headline(article, keywords) as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;

Will give you:

	:::text
	   title     |                                                      result                                                      
	--------------+------------------------------------------------------------------------------------------------------------------
	Tiny Allan   | <b>gently</b> rapping, rapping at my chamber <b>door</b>. "Tis some visitor," I muttered, "tapping at my chamber
	Small Allan  | <b>gently</b> rapping, rapping at my chamber <b>door</b>. "Tis some visitor," I muttered, "tapping at my chamber
	Medium Allan | <b>gently</b> rapping, rapping at my chamber <b>door</b>. "Tis some visitor," I muttered, "tapping at my chamber
	Big Allan    | <b>gently</b> rapping, rapping at my chamber <b>door</b>. "Tis some visitor," I muttered, "tapping at my chamber


As you can see, we get back a short excerpt of each verse with the tokens of interest surrounded with a HTML "<b\>" tag.
That is actual all there is to this function, it return the results with the tokens highlighted.

However, there are some nice options you can set to alter this basic behavior.

The first one up is the HTML tag you wish to put around you highlighted words. For this you have two variables *StartSel* and *StopSel*.
If we wanted this to be a "<em\>" tag instead, we could tell the function to change as follows:

	:::sql
	SELECT title, ts_headline(article, keywords, 'StartSel=<em>,StopSel=</em>') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;
		
And now we will get back an <em\> instead of a <b\> (including just one row this time):

	:::text
	   title     |                                                        result                                                        
	--------------+----------------------------------------------------------------------------------------------------------------------
	Tiny Allan   | <em>gently</em> rapping, rapping at my chamber <em>door</em>. "Tis some visitor," I muttered, "tapping at my chamber

In fact, it does not need to be HTML at all, you can put (almost) any string there:

	:::sql
	SELECT title, ts_headline(article, keywords, 'StartSel=foobar>>,StopSel=<<barfoo ') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;

Result:

	:::text
	   title     |                                                               result                                                               
	--------------+------------------------------------------------------------------------------------------------------------------------------------
	Tiny Allan   | foobar>>gently<<barfoo rapping, rapping at my chamber foobar>>door<<barfoo. "Tis some visitor," I muttered, "tapping at my chamber

Quite awesome!

Another attribute you can tamper with is how many words should be included in the result set by using the *MaxWords* and *MinWords*: 

	:::sql
	SELECT title, ts_headline(article, keywords, 'MaxWords=4,MinWords=1') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;

Which gives you:

	:::text
	   title     |             result             
	--------------+--------------------------------
	Tiny Allan   | <b>gently</b> rapping, rapping

To make the resulting headline a little bit more readable there is an attribute in this options string called *ShortWord* which tells the function which is the shortest word that may appear at the start or end of the headline. 

	:::sql
	SELECT title, ts_headline(article, keywords, 'ShortWord=8') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;
		
Will give you:

	:::text
	   title     |                                                                                   result                                                                                    
	--------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
	Tiny Allan   | <b>gently</b> rapping, rapping at my chamber <b>door</b>. "Tis some visitor," I muttered, "tapping at my chamber <b>door</b> - Only this, and nothing more."
	Small Allan  | <b>gently</b> rapping, rapping at my chamber <b>door</b>. "Tis some visitor," I muttered, "tapping at my chamber <b>door</b> - Only this, and nothing more." Ah, distinctly

Now it will try and set word boundaries to words of minimal 8 letters. This time I included the second line of the result set. As you can see the engine could not find an 8 letter word at the remainder of the document, so it simply prints it until the end. The second row, "Small Allan" is a bit bigger and the word "distinctly" has more then 8 letters, so is set as the boundary,

So far the headline function has given us almost full sentences and not really fragments of text. This is because the optional *MaxFragments* defaults to 0. If we up this variable, it will start to include fragments and not sentences. Let us try it out:

	:::sql
	SELECT title, ts_headline(article, keywords, 'MaxFragments=2,MaxWords=8,MinWords=1') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;

Gives you

	:::text
	   title     |                                                     result                                                      
	--------------+-----------------------------------------------------------------------------------------------------------------
	Tiny Allan   | <b>gently</b> rapping, rapping at my chamber <b>door</b>
	...
	Big Allan    | <b>gently</b> rapping, rapping at my chamber <b>door</b> ... chamber <b>door</b> - This it is, and nothing more

I include only the first and last line of this result set. As you can see on the last line, the result is now fragmented, and we get back different pieces of our result.
If, for instance, four or five tokens match in our document, setting the *MaxFragments* to a higher value will show more of these matches glued together.

Accompanying this *MaxFragments* option is the *FragmentDelimiter* variable which is used to define, well, the delimiter between the fragments. Short demo:

	:::sql
	SELECT title, ts_headline(article, keywords, 'MaxFragments=2,FragmentDelimiter=;,MaxWords=8,MinWords=1') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;
		
You will get:

	:::text
	   title     |                                                   result                                                    
	--------------+-------------------------------------------------------------------------------------------------------------
	Big Allan    | <b>gently</b> rapping, rapping at my chamber <b>door</b>;chamber <b>door</b> - This it is, and nothing more

Including only the last line, you will see we now have a semicolon (;) instead of a ellipses (...). Neat.

A final, less common option for the *ts_headline()* function is to ignore all the word boundaries we set before and simply return the *whole* document and highlight all the words of relevance.
This variable is called *HighlightAll* and is a Boolean set to *false* by default:

	:::sql
	SELECT title, ts_headline(article, keywords, 'HighlightAll=true') as result
		FROM phraseTable, to_tsquery('door & gently') as keywords
		WHERE phrase @@ keywords;

The result would be too large to print here, but try it out. It will give you the whole text, but with the important tokens decorated with the element (or text) of choice.

### A big word of *caution*

It is very fun to play with highlighting your results, I will admit that. The only problem is, as you might have concluded yourself, this is a potential performance grinder.

The problem here is that this function cannot use any indexes and it can also not use your stored tsvector. It *needs* the original document text and it needs to not only parse the whole document text to a tsvector for matching, it also needs to parse the original document text a second time to find the substrings and *decorate* them with the characters you have set. And this whole process has to happen *for every single record* in your result set.

Highlighting, with this function, is a *very* expensive to do. 

This does not mean that you have to avoid this function, if so I would have told you from the start and skipped this whole part. No, it is there to be used. But use it in a correct way.

A correct way often seen is to use the highlighting only on the top results you are interested in - the top results the user has on their screen at the moment.
This could be achieved in SQL with a so called *subquery*.

	:::sql
	SELECT title, ts_headline(article, keywords) as result, rank
		FROM (SELECT keywords, article, title, phrase, ts_rank(phrase, keywords) AS rank
			  FROM phraseTable, to_tsquery('door & gently') AS keywords
			  WHERE phrase @@ keywords
			  ORDER BY rank
			  LIMIT 2) AS alias;

For those unfamiliar, a *subquery* is nothing more than a query within a query (queue Inception drums...sorry).

You evaluate the inner query and use the result set of that to perform the outer query. You can achieve the same with two queries, but that would prove not to be as elegant.
When PostgreSQL sees a subquery, it can plan and execute more efficiently then with separate queries, many times giving you a better performance.

The query you see above might look a bit frightening to beginning SQL folk, but simply see it as two separate ones and the beast becomes a tiny mouse.
Unless you are afraid of mice, let it become a...euhm...soft butterfly gliding on the wind instead.

In the inner query we perform the actual matching and ranking as we have seen before. This inner query then only returns *two* matching records, because of the *LIMIT* clause.
The outer query takes those results and performs the expensive operation of highlighting.

## Indexing

Back to a more serious matter, the act of *indexing*.

If you do not know what an index is, you have to brush up real fast, for indexing is quite important for the performance of your queries.
In a very simplistic view, an index is like a chapter listing in a book. You can quickly skim over the chapters to find the page you are looking for, instead of having to flip over every single page.

You typically put indexes on tables which are consulted often and you build the index in a way that is in parallel with how you query them.

As indexing is a whole topic, or rather, a whole profession of its own, I will not go too deeply into the matter.
But I will try to give you some basic knowledge on the subject.

Note that I will go over this matter in lighting speed and thus have to greatly skim down on the amount of details.
A *very* good place to learn about indexes is Markus Winand's [Use The Index, Luke](http://use-the-index-luke.com/ "Use The Index, Luke series written by Markus Winand.") series.
I seriously suggest you read that stuff, it is golden knowledge for every serious developer working with databases.

### B-tree

Before we can go to the PostgreSQL full text index types we first have to look at the most common index type, the *Balanced tree* or *B-tree*.

The *B-tree* is a proven "computer science" concept that give us a way to search certain types of data, fast.

A *B-tree* is a tree structure with a root, nodes and leafs (inverse from a natural tree).
The data that is within your table rows will be ordered and chopped up to fit within the tree.

It is called *Balanced*, meaning that each level of the tree has the same amount of nodes.

Take this picture for example:

	:::bash
	            |root|
                  |
           ----------------
           |               |
        |node|          |node|
	       |               |
      ----------       --------- 
      |        |       |       |
    |node|  |node|  |node|  |node|


In *B-tree* terms, we summarize this tree by saying:

* It has an *order* of 2, meaning that each node will contain two leaves only
* It has a *depth* of 3, meaning it is three levels deep (including the root node)
* It has 4 *leaves*, meaning that the amount of nodes that do not contain children is 4 (bottom row)

If you set the *order* of your tree to a higher number, more nodes can fit onto a single row and you will end up with a lesser *depth*.

Now the actual power of an index comes from an I/O perspective. As you know (or will know now) the thing that will slow down a program/computer the most is I/O.
This can be network I/O, disk I/O, etc. In case of our database we will speak of disk I/O. 

When a database has to go and *scan* your table without an index is has to plow through all your rows to find a match.
Database rows are almost always *not* I/O optimized, this means that they do not fit well in the blocks of your physical disks structure.
This, in short, means that there is a lot of overhead in reading through that physical data,

A *B-tree* on the other hand, is *very* optimized for I/O. Each level of a *B-tree* will try and fit perfectly within one physical block on your disk.
If all levels fit within one block each, walking over the tree will be very efficient and have almost no overhead.

*B-trees* work with the most common data types such as TEXT, INT, VARCHAR, ... .

But because full text search in PostgreSQL is its own "thing" (using the @@ operator), all knowledge that you may have learned about regarding indexes does not apply (or not in full anyway) to full text search.

Full text search needs its own kind of indexing  for a *tsquery* to be able to use them. 
And as we will see in a moment, indexing on full text in PostgreSQL is a dance of trade-offs.
When it comes to this matter we have two types of indexes available: *GiST* and *GiN* which are both closely related to the *B-tree*.

### GiST

GiST stands for *Generalized Search Tree* and can both be set on *tsvector* and *tsquery* column types, though most of the time you will use it on the former.

The *GiST* itself is not something that is unique to PostgreSQL, it is a project on its own and its concept is laid out in a C library called *libGist*.
You could go ahead and play around with *libGiist* to get a better understanding of how it works, it even comes shipped with some demo applications.

Over time there have come many new types of trees based on the *B-tree* concept, but most of them are limit in how they can match.
A *B-tree* and its direct descendants can only use basic match operators like "<", ">", "=", etc.
A *GiST* index, however, has more advanced matching capabilities like "intersect" and in case of PostgreSQL's implementation: the *"@@"* operator.

Another big advantage of the *GiST* is the fact that it can store arbitrary data types and therefor can be used in a wide area of conduct.
The trade off for the wide data type support is the fact that *GiST* will always return a *no* if there is no match or a *maybe* if there is.
There is no true *hit* with this kind of index.

Because of this behavior there is extra overhead in the case of full text search because PostgreSQL has to manually go and check all the *maybe*'s that return and see if they are an actual match.

The big advantages of *GiST* are the fact that the index builds faster and the update of such an index is less expensive then the next index type we will see.

### GiN

The second index candidate we have at our disposal is the *Generalized Inverted Index* or *GiN* in short.

Same as we saw with *GiST*, *GiN* also allows for arbitrary data types to be indexes and allows for more matching operators to be used.
But as opposed to *GiST*, a *GiN* index is *deterministic* - it will always return a true match, cutting the checking overhead needed with *GiST*.

Well, unless you wish to use *weights* in your queries. A *GiN* index does *not* store lexeme weights. This means that, if weights need to be taken into account when querying, PostgreSQL still has to go and fetch the actual row(s) that return a true match, giving you somewhat of the same overhead as with a *GiST* index.

*GiN* tries to improve the *B-tree* concept by minimizing the amount of redundancy within nodes and there leaves.
When you search for a number between 0 and 1000, it can be that your index has to go over 5 levels to find the desired entry.
This means that the four levels above the matching leaf could potentially contain an (implied) reference to the row id you wish to have fetched.
In a *GiN* index, this is generalized by storing a single entry of the duplicates into so-called *posting trees* and *posting lists* and pointing to those lists instead of drilling down multiple levels.

The downside of *GiN* is the fact that this kind of index will slow down the bigger it gets.

On a more positive note, *GiN* indexes are most of the time smaller on disk (because of it trying to reduce duplicates). And, as of PostgreSQL 9.4, they will be *even* smaller.
The soon-to-be version will introduce a so-called *varbyte* version of *GiN*. For now just take it from me that it will make these type of indexes *much* smaller, and even more efficient.

As you can see, there is no perfect index when it comes to full text. You will have to carefully look at what data you will save and how you wish to query the data.

If you do not update your database much but you have a lot of querying going on, *GiN* might be a better option for it is much faster with a lookup (if no weights are required).
If your data does not get read much, but is updated frequently, maybe a *GiST* is a better choice for it allows for faster updating.


### Making an index

We have (very roughly) seen what an index is and what we have available for full text, but how do you actually build such an index?

Luckily for us, this too has been neatly abstracted and is very simple to do.

If we wanted our phraseTable to contain an index, we simply could go about and create it with the following syntax:

	:::sql
	CREATE INDEX phrasetable_idx ON phraseTable USING gin(phrase);

This will create a *GiN* index called *phrasetable_ixd* on the column *phrase*.

Just like we did before, we will now re-populate our *phrase* column, but this time we will fill it with the data we want to have indexed: article and title.
Let me show you what I mean.

First, empty the four *phrase* columns in our tiny database:

	:::sql
	ALTER TABLE phraseTable ALTER phrase DROP NOT NULL;
	UPDATE phraseTable set phrase = NULL;

Notice that I removed the *NOT NULL* constraint.
Next we can populate it containing a tsvector of both the *title* and the *article* columns:

	:::sql
	UPDATE phraseTable SET phrase = to_tsvector('english', coalesce(title,'') || ' ' || coalesce(article,''));

The *coalesce* function may be something that you are unfamiliar with.
This functions simply returns the first argument which is *not* NULL.
In this case we use:

	:::sql
	coalesce(title,'')

Which means that if title would be NULL it will return the empty string *''* which never is NULL.
We use *coalesce* here to substitute a value for NULL, being the empty string.

If we would not substitute NULL then our *tsvector* generation would fail if either the title or article column would be NULL.

Next we can create an index on that newly filled column:

	:::sql 
	CREATE INDEX phrasetable_idx ON phraseTable USING gin(phrase);

And we have magic, there now is a *GiN* index on that column which will be used during full text search.

To create a *GiST* index we could use exactly the same syntax:

	:::sql 
	CREATE INDEX phrasetable_idx ON phraseTable USING gist(phrase);

Now, the disk-space savvy readers will have noticed that our "phrase" column now contains some redundant information as we store the tsvector of the article and title column that was already in the database.
If you do not wish to have this extra column, you could created *expression* indexes (our on-the-fly queries we seen before).

The setup of such an expression index is trivial:

	:::sql
	CREATE INDEX phrasetable_exp_idx ON phraseTable USING gin(to_tsvector('english', coalesce(title,'') || ' ' || coalesce(article,'')));

Instead of having this extra tsvector column around, we now have created an on-the-fly index using the same syntax as we employed when we populated the *phrase* column a few lines back.

One important thing to note when you use *expression* indexes is the text search configuration you used. Here we specify that we wish the index to be created using the 'english' configuration set.
This results in an index which is configuration aware and will *only* work with a query which has the *same* configuration set fed to the tsquery function (well, the same name anyway).

You could omit the configuration which would then default to the one set in the "default_text_search_config" variable we saw in the last chapter. The problem you will have then is that the index is created using a configuration that *could* be altered *after* the index was created. If we later would query the database with the altered default, the index would be useless and will return inaccurate results. 

Also note that we may save on disk space when we use the *expression* index, but we do not save on CPU. Now, instead of indexing data already parsed and ready in a column, the index has to compute the *to_tsvector* on every index match. Again, a world of trade-offs.

## Triggers

A final, small topic I want to briefly touch on before I let you go free are *update triggers*. The way we have been populating our database so far does not need a trigger actually. Up until now we have been inserting records (or updating them) using the *ts_tsvector()* function. The negative aspect of going about it the way we did is that it is extremely *redundant*. 

If we inserted a piece of Raven text into a record, we specified it *twice*, one time for the *article* column and one time for the *phrase* column which holds the tsvector result.

A better way to do this is to not let the insert query care about the tsvector *at all*. We simply insert the text we like and let the database do the converting  behind the curtains.

This is where a *trigger* comes in handy. Actually, PostgreSQL has a whole set of *trigger* functions available that will fire when certain conditions are met, but when it comes to full text we have two functions at our disposal.

### tsvector_update_trigger()

The first, and most used one, is called *tsvector_update_trigger()* and fires whenever a new row is inserted into your table (in our case *phraseTable*).

To setup such a trigger, we could use the following SQL:

	:::sql
	CREATE TRIGGER tsvectorupdate BEFORE INSERT OR UPDATE
		ON phraseTable FOR EACH ROW EXECUTE PROCEDURE
		tsvector_update_trigger(phrase, 'pg_catalog.english', title, article);

That is all you need to setup such a trigger. Let us see what we just did.

First, we have new syntax staring us in the face: *CREATE TRIGGER*. This will create a trigger on certain *events*. The events here are *BEFORE INSERT* and *BEFORE UPDATE* which are contracted to *BEFORE INSERT OR UPDATE*. Then we specify on which *table* this trigger has to act and for each *ROW*. Then we say we want to *EXECUTE PROCEDURE*, which, in our case, is the function *tsvector_update_trigger()*.

The function itself needs a bit of explaining as well. This version takes three required arguments: the tsvector column name, the full text configuration name and the original text column name.
The latter can be multiple columns to concatenate them together. This concatenation is done with *coalesce* under the hood, as we have seen before.

In our case, we create a trigger that takes the *phrase* tsvector column, the *english* full text configuration and concatenates the text from both *title* and *article* to be normalized into lexemes.

Note that instead of *english* we say *pg_catalog.english* when providing this function with the full text configuration.
In case of this function (and the next) we have to provide the schema-qualified path to the configuration.

### tsvector_update_trigger_column()

The other of the two full text trigger functions we have is called *tsvector_update_trigger_column()* and has only one difference to the former: the full text configuration used.
Here, the full text configuration can be read from a *column* instead of given directly as a string.

A possibility we have not seen in this series is one where you can have yet another column in your phraseTable where you store the name of the full text configuration you wish to use.
This way you can store multiple "languages" within the same table, specifying which configuration to use with each row.

This trigger functions can take into account these per-row differing configurations and is able to read them from the specified column.

But we have a trade-off once more. These two trigger functions, which are officially called *example functions* again (remember our ranking functions?), do *not* take into account weights.
If you have the need to store different weights in your tsvectors, you will have to write you own trigger function.

## The end

Okay, I guess this covers the *basics* of full text within PostgreSQL.

We have covered the most important parts and touched some segments deeply, others just with a soft lovers glove.
As I always say at the end of such lengthy chapters: go out and explore.

I have tried to give you a solid, full text knowledge base to build further adventures on. I highly encourage you to pack your elephant, take your new ship for a maiden voyage, set high the sails and if certain blue wales try to swim next to your vessel, simply let the mammoth take a good relief down the ship's head, and let those turds float together with our squeaky finned friends!

And as always...thanks for reading!

<!--  LocalWords:  PostgreSQL lexeme
 -->
