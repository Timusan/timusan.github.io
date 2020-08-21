<!--
.. title: The Scheme programming language AKA The CHICKEN hens nest - Part 1
.. slug: chicken-scheme-1
.. date: 2013/08/23 19:30:00
.. tags: chicken, scheme, json rpc
-->

## Chapter 1 - A Schemey world of wonder

I just recently finished writing my first egg (module) for the Scheme implementation called CHICKEN.
For me, this seemed as a good point in time to take you on a journey to the world of the Scheme programming language.

The goal of these posts is to show you how to write your own egg for CHICKEN.
Or, in other words, how to contribute and in turn learn about this dialect.

I assume that you have a basic notion of computer programming and know how to use a Unix system on a basic level.
All the rest we can explore in these three chapters!

**Beware:** I will fire a lot of information in your general direction...so if you don't understand something, allow yourself to take a step back.
Put these posts down and pick them up a day later. Don't push yourself to read something if you did not understand the part that came before it.

### So what will we see in this first chapter?

* A preface
* Another preface
* What is CHICKEN?
* Draw an outline of the egg we will make
* Looking at (CHICKEN) data types
* Using Medea, an egg for CHCIKEN <-> JSON translation
* Learning about cons cells
* What are car, cdr and cons?
* Start to hatch our egg
* Looking at the skeleton of a CHICKEN egg
* Using predicates
* Looking at exception handling
* Take a deep breath and look at what we have seen
* Prepare for chapter 2

## Preface

Imagine me and Bill (also known as Bill the Brogrammer) walking down a street on a rainy evening.
Next to us, there is something else walking, a stranger, an unknown.
All of a sudden I turn to Bill and say: "Ever heard of LISP or Scheme?".

Bill gives me a blank stare.

"No? Well, then it's time that I introduce you to each other!"
I take a step back so he can see our stranger and shake its hand. 
"Bill, this is Scheme! Scheme, this is Bill!"

But Bill frowns and says: "It looks strange, almost from another planet...and...it looks like my grandfather."

"Now, now, no need to judge so early!", I say, "True that it may look a bit uncommon, and true that it's older then the pilot episode of Lost in Space, but that does not mean a thing!".

Bill looks closer into it's luminescent little eyes and states in a monotone voice: "If I befriend it, people will frown upon me. People will think that I'm smug, a weeny, something better then they are, they will hit me with sticks and chaise me with pitchforks...".

"All of that...just because you have befriended Scheme?", I ask a bit confused.

"Yes! If I come too close to it, I may loose interest in my other friends, especially my good friends Pee-hey-pee<sup>1</sup> and Jaabaa<sup>1</sup>."

I snuggle ever so slightly: "What did *those* friends ever do for you?"

"*Those* friends gave me scripts, a corporate look...and...a shiny bandwagon to ride on together with all my other Brogrammers...", Bills eyes turn watery.
With a small voice he asks: "Does Scheme have a shiny bandwagon too?".

I shake my head: "No, I'm sorry, it does not. But it has something else..."

The Schemy little thing puts out its bony hands, holding out a small box.

"Take the box", I instruct Bill, "Take it and open it up.".

Full with doubt Bill takes the box and slowly opens it. His face becomes lit with a bright, angle-like light. A street choir starts to sing.
From the box rises and ancient figure, elegant, light as a feather. Bill starts to weep, tears streaming from his eyes.
"Its....its...the Lambda!", he bursts out in tears of joy, "I see the light...I see it now...its so beautiful!".

"Yes it is, yes it is." I pat Bills shoulder and the three of us walk of into the moonlight.

*<sup>1</sup><small>fictional languages, any resemblance to real languages is pure coincidence...</small>*

## Less cheesy preface

Okay, that last part may have been a little bit over the top...

But before we dive into Scheme I want to take away the strange light that this language and its users are being put in by some people.
It is true that Schemers (people using Scheme) are smug-lisp-weenies (or smug-scheme-weenies or even smug-list-weenies, ha!), but that is not a *bad* thing.

From my personal experience, Schemers have been the best and most competent programmers that I have worked with so far.
And yes, they can be smug, really smug, but only when it is *justified* to be so. When another language fails horribly, Scheme most of the time has a much better alternative.
A *good* Schemer is smug by pointing out how it can be done better, safer and faster. A *good* Schemer will never, ever be smug on a personal level.

Scheme, or it's ancestor LISP, is a language that is designed around a totally different view then most, current programming languages.
It has been around long enough to see the birth of all the current hip and cool languages.
And it goes back all the way to the 1950's, quite close to the birth of the modern day computer. *And it is still here.*

Before I start sounding like and evangelist who should start a Church Of Scheme, there are, of course, other nice and elegant languages.
To name a few: Python, Erlang, Pascal, ... . There are dozens of decent languages out there.
Using Scheme does not exclude you from using other languages or vice versa. It will, however, teach you more about *real* programming.
If used correctly, Scheme can teach you good practices that you can also use (to a certain extend) in other programming languages.

It's a *minimalistic* language with an almost human readable notation, without almost any syntax hassle and which invites you, no, *encourages* you to contribute to extend its functionality.

Good, now that we have that off the table, its time to get our hands dirty!

## What is CHICKEN?

First things first, what is this CHICKEN I keep referring to?

To understand this we have to take a look at how Scheme is used to form various "dialects".
Scheme itself is rarely found, most of the time it is an interpretation of a *specification* that you will use to program in.

These so-called dialects are based on specifications that aim at making the different Schemes lightweight languages. This lightweight aspect is in contrast to *Common Lisp* (Schemes twin survivor of the old LISP mother) which includes everything and the kitchen sink.

The specifications that are used *as guidelines* to build the various Scheme dialects are called the R*n*RS specs. And just recently, the first draft for the latest spec (R7RS) was released.

Because *anyone* with some degree of Scheme knowledge can build its own implementation, there are literally thousands of dialects.
Most of them never make it out of the developers computer though, as they are mere experiments to get to know the language better.
But some do make it out and CHICKEN is one such dialect.

Conjured up by [Felix Winkelmann](http://spin.atomicobject.com/2013/05/02/chicken-scheme-part-1/ "Reddit article about Felix Winkelmann") in 2000, CHICKEN is a Scheme implementation based on the R5RS spec.
It is one of the few Schemes out there with a very rich ecosystem of supporting modules (from now on referred to as *eggs*) that extend CHICKEN's core functionality.
And in my personal experience, the CHICKEN community consists of very, very friendly people who are always willing to help, whether you are an absolute nooby or a seasoned Schemer.

## How do you "get" CHICKEN?

Before we can start programming using CHICKEN, we have to install it first.
This can be done on a [wide variety of platforms](http://wiki.call-cc.org/platforms "CHICKEN WIKI entry for installing on different platforms.") ranging from Linux, BSD, OSX and even Windows, but for the scope of these posts I will be using Linux.

On most Linux systems there are binaries available that you can use to get up and running quickly.

I use Arch Linux as my main development platform, so I will stick to that for setting everything up.
To install CHICKEN you can simply let your package manager do the heavy lifting:

	:::bash
	pacman -S chicken

That's it! You now have a fully functional CHICKEN system.

There are three programs that have been installed that you should take note of:

* csc: The *C*HICKEN *S*cheme *C*ompiler
* csi: The *C*HICKEN *S*cheme *I*nterpretor
* chicken-install: The chicken egg installer

Let us go over them briefly.

### csc

This is the *CHICKEN compiler* and is used to "convert" or compile Scheme code into raw C.

*What is compiling, you ask?*

Before the day of C, PASCAL or FORTRAN (which are all referred to as "low level" programming languages) you had to completely rewrite you code for every different architecture that you wanted your program to run on. This was a big, time consuming hassle and the computing industry needed something more flexible, something more *portable*. A *generic* language that programmers could use. The C programming language was born. Well, to be more accurate, FORTRAN was first, and is considered the oldest language still in use. But because FORTRAN is more in the field of scientific computing, C became a more popular general purpose language to program in.

This introduced a more fluid way of working but also introduced an extra step before you could run your portable code on a specific architecture.
This step is called *compiling*. Because the different chipsets still needed their own, specific instructions the code still needed to be converted into their supported *binary* format.

To this day of age, this principle is unchanged. We are still stuck to the same multi architectural differences. Computers haven't actually evolved much since the old days.

Because by modern day standards C itself isn't the most practical, friendly nor safe language to program in, the same idea happened here as well.
So called "high level" programming languages where build on top of C (or along side C) that where more soothing to program in.

And yet another step was introduced.

Before these high level languages could be run by the computer, most of them have to be compiled to portable C code which, in turn, compiles it to architecture specific binary code.
The compilation to C is, in most cases, necessary because the C compiler is one of the few compilers that knows almost *every* dirty, nitty gritty detail about every chipset, every instructions set available out there. If you want to cut out the C compiler and compile directly to binary code, you will have to implement *every goddamn* chipset yourself. Since this is generally considered the work of a mad man, people tend to use the C compiler in the final step to spit out binary code.

And that is exactly what happens with the CHICKEN compiler. You compile your Scheme code to portable C code, and then use a C compiler (usually GCC) to compile to a binary file.

What is interesting for us in this stage is the optimization that happens before *csc* compiles the portable C code.
This is an in-between step that will produce a more optimized version of your CHICKEN code so that the compiler can make more smart decisions.
The optimization step gives as a nice report at the end of the optimized CHICKEN file which we can read and use to "clean up" our source code.

Don't worry about this too much, it will be more clear when we actually come to this step in chapter 2.

### csi

This is the *CHICKEN interpreter* and is used to quickly throw some code at and see the results without the need to compile everything.
Another name for this is a *REPL*, which is short for *R*ead, *E*valuate and *P*rint *L*oop:

* You punch in a line of code that it *reads*
* The interpreter goes and *evaluates* that code
* It *prints* the result of that evaluation
* Then it goes back into read mode, waiting for your input and thus making a *loop*

You will be using the REPL a lot when developing. Keep in mind though that the interpreter is considerably slower then when you compile your code. So, in CHICKEN at least, its perfect for development purposes, but not for use in "the real world". 

### chicken-install

This is the *CHICKEN Egg installer* which you can use to...well...install eggs and extend the functionality of your local CHICKEN install.

## Outlining the Egg

Okay, now that we have the basics up and running on our system, its time to build a clear picture of what we want to make.

In my case, I wanted to build an egg that would format user input (the user here being the programmer using this egg) into a JSON-RPC 2.0 valid string.

If you don't know what JSON is, keep reading!

I found that there was no such egg available in the [egg index](http://wiki.call-cc.org/chicken-projects/egg-index-4.html "Index of CHICKEN v4 eggs"), so I decided to write it myself and do a contribution to the CHICKEN ecosystem.

This egg is actually part of a larger plan I have in mind, that is, to control my XBMC media box from a terminal application.
In my mind, this larger plan would consist out of three eggs:

* *A JSON-RPC translator* (the one we will build now):
   The XBMC system uses JSON to let other devices communicate with and control it. So on a basic level we need the instructions that we sent to be valid JSON-RPC.

* *A XBMC API in CHICKEN*:
   XBMC has an elaborate API, in JSON of course, that we have to abstract into Scheme. This is done with the help of the JSON-RPC egg.

* *An end-user program*:
   A program, which implements the above XBMC in Scheme API, that the end-user can actually use to control their XBMC box.

But for now, you can forget about the last two and focus on the JSON-RPC egg.

*What is JSON?*

JSON is an acronym for *Javascript Object Notation* and is a standardized way to exchange textual information between systems. It's as simple as that.
It uses the JavaScript language representation of data and puts in it a serialized form we call a string.

*Then, what is a JSON-RPC string?*

JSON-RPC is a [specification](http://www.jsonrpc.org/specification "JSON-RPC version 2.0 specification") that standardizes a way of invoking RPC's or *Remote Procedure Calls*. RPC is nothing more then a fancy term which means the same as a way to start a "procedure" on another machine.
Because we are dealing with data *exchange* we have a client and a server part:

* Client:
  The client can send a *request* object to the server
* Server:
  The Server can send back a *response* object if the request object was understood and an *error* object if the request was in error.

Since the JSON-RPC egg that I made only focuses on the client side (for now) we can concentrate on the *request* object only.
According to the spec the request string consists out of four different entities where one entity is optional:

* A string containing the JSON-RPC *version number*
* A string containing the *method*
* An *optional* array (named or positional) containing the *parameters* (called params)
* A unique *ID*

So a JSON string build according to this spec would look something like this:

	:::json
	{"jsonrpc": "2.0", "method": "subtract", "params": [42, 23], "id": 1}

What we want our egg to do is to accept, in some form, some data (version, method, params and id), construct this JSON string and finally output it.

But how?

## (CHICKEN) Data types

CHICKEN does not output JSON, even more so, CHICKEN does not *know* JSON. To CHICKEN, JSON is just a string containing a bunch of characters.
So to be able to work with JSON inside of CHICKEN, it's important that we think of a way we can represent JSON in CHICKEN.
This representation is done trough CHICKEN data types.

*Data types?*

Every programming language has data types. They are vital building blocks. A data type is a defined *container* which can hold a specific type of data.
The data type *number* (in Scheme), for example, can only hold integers or the data type *boolean* can only hold true or false.
They tell the system something about the data they contain.

In the world of programming languages we have two data type systems:

* Dynamically typed (Scheme (CHICKEN), Python, Erlang, JavaScript, Ruby ...)
* Statically typed (C, Haskell, Pascall ...)

There is *great* discussion as to which type system is better (not to mention the subtypes "weak" and "strong").
I'll not go into the hairy discussion (for it is unshavably hairy), let us just settle by saying that both systems have their advantages.

With a *strong dynamically* typed language like CHICKEN, the type is added to the value and not to the variable itself. So you can put a string into a variable that held a number before, but you cannot, for instance, add up a number type (1) and a string type (containing the character 1). It catches programming errors early (during compilation) and puts in a layer of security (data and data types cannot be mixed up). 

For now I would be happy if you just remember there are two types and CHICKEN is of the dynamic kind.

So what we have to do next is look at which data types we can use to represent a JSON-RPC valid string in Scheme.

## Medea, the JSON egg

As we have seen before JSON is a JavaScript based textual representation of data which means that JavaScript is the only language where a JSON string *does* make sense.
Actually, a JSON string contains JavaScript data types!

Having this info we don't have to figure out everything by ourselves, we just need to make a mapping from JavaScript data types to CHICKEN data types and we have our representation.

Simple, right? Even more simple with an egg called [Medea](http://wiki.call-cc.org/eggref/4/medea "The Wiki entry for the Medea egg.").

This egg does *exactly* what we described above: map JSON (JavaScript) data types to CHICKEN data types and the other way around.

Whoohoo!

But Medea does a conversion for almost *every* data type, so first we have to look at which data types we actually need.
Since our starting point is an existing JSON-RPC string we have to start by looking at the JavaScript side of things.
Let us introduce a XBMC JSON-RPC string to dissect:

	:::json
	{"jsonrpc": "2.0", "method": "Player.PlayPause", "params": { "playerid": 0 }, "id": 1}

In this string we find two different JavaScript data types:

* A string
* An object

Not too shabby, hey? Just two data types we have to concern ourselves with. If we feed these two data types to Medea, what will it spit out?

* A string becomes a *string*
* An object becomes a *associative list* or *alist* in short.

Also, not too much to handle.

Actually, according to the JSON-RPC spec, we can have an object *or* an array.
The object can be used to give *named* params (as in the example above), the array can be used to gives *positional* params. In CHICKEN a *positional* array can be translated to a *vector*

So we actually have three data types to look at. Still not too bad!

Let us look at each of them and see what they look like in CHICKEN.

A string in CHICKEN looks like this:

	:::scheme
	"This is a string"

Just the same as in most languages. A sequences of characters wrapped in double quotes.
On to the next one, a vector (this would be a normal array in another language):

	:::scheme
	#("foo" "bar")

This is a vector containing two strings. It is represented as a list proceeded with a hash symbol.
And finally an associative list (this would be a hash table or associative array in another language):

	:::scheme
	((foo . bar) (faa . bor))

Also, not too scary, just a list containing pairs.

*Waah, wait, what are all these parenthesis doing there? And what are these lists you speak of?*

Phew, I thought you would never ask!

These lists and the parenthesis that represent them are a **very** important part of the language syntax.
This is what many programmers (who don't use a Scheme) see as difficult and sometimes define it as incomprehensible.
If you show a piece of Scheme code to a non-Schemer, many time they let out a girly shriek, scared by the hundreds of parenthesis crawling over their screen.

And most of the time their first reaction (after the shriek) is: "How can you even read this stuff?".

The answer is simple: You just *do*.

In the beginning you will find this strange and maybe less clear to read. But the more you *actually program* with a Scheme, the more this syntax will start to feel natural.
And eventually you will feel that these lists and parenthesis will free you from the syntax horror that some other languages can bestow upon you.

The reasoning behind these lists lies in history. In the beginning of the post, we talked about Scheme being a remainder of the old LISP mother.
LISP, which is short for *LIS*t *P*rocessing, is a language developed in the 1950's by John McCarthy at the MIT University.

In this language (and all the successors) everything is a list (that not entirely true, but let us forget that for now).
In the beginning lists where simply the way this language represented data, but as the language grew more mature, lists quickly proofed to give an excellent solution to something LISP languages are very good at.

What is that you ask? This: *code is data and data is code*.

I will not be going deeper into this right now as it is beyond the scope of these posts, but just know that this is one of the reasons why Schemers love their parenthesis.

Oh, and one other important thing, Schemers like to refer to lists as *s-expressions*. Well, actually *lists* are *s-expressions* but *s-expressions* don't have to be *lists*. In Scheme *symbols*, *vectors*, *strings*, ... are all *s-expressions*.

Okay, after this very brief run trough history, let us turn back to our data types we just saw.

We have concluded that we want to use the Medea egg to translate our CHICKEN data types in JSON and we have determined which data type we want to use.
The next logical step is too see how we can actually use Medea inside of CHICKEN to do the conversion for us.

Its time to use...the *REPL*!

Yes, after all this talk, we can finally get our hands dirty!
As we have seen before, the REPL in CHICKEN is called *csi*, so open up a terminal and punch the command *csi*.

You will see a small greeting and a blinking cursor waiting for your input.
By default, the interpreter has no extra modules loaded. If we want to use Medea, we first have to load in that module.
So punch in this line and hit enter:

	:::scheme
	(use medea)

And we have....an error. Ah, aren't I a sneaky bastard...CHICKEN does not know this module "Medea" that you want to use. Why? Because we haven't installed it yet.
So leave you interpreter by typing:

	:::scheme
	.q

Before you can install any egg, you have to be root.
So become root and install Medea:

	:::bash
	chicken-install medea

Now switch back to your user and start the interpreter again.

It's important to know that when you exit the interpreter it will reset and forget everything you have done in that session.
So you have to load in Medea again. Now it should show a bunch of lines telling you want it imported and return back to read mode.

Every egg in CHICKEN comes with documentation, so its important you read it carefully before using it or asking help on mailings lists or IRC.
According to Medea's documentation we have two procedures we can use to read or write JSON.

*What's a procedure?*

A procedure can be compared with a function in another language. It's a bunch of lists that do something, usually return a value.

Small note about *lists*: I just told you a little lie, I said that procedures are lists, but actually, in CHICKEN, they are only *represented* as lists and are their own thing entirely when compiled. There is a common misconception that in CHICKEN or Scheme everything is a list. This comes from the old LISP, where this *was* true. However, one of the things that separates Scheme from LISP is the very fact that *not everything* are lists anymore. Just keep a mental note of that.

Okay, let us continue.

By loading in the egg we have many new procedures at our disposal.
There are two of them which we can use directly in the REPL to see the conversion in action:

	:::scheme
	(read-json datum)
	(write-json datum)

*Datum* is a placeholder used to describe any form of data that can be put there.
Let us take the JSON string we dissected above and see what Medea makes of it.

	:::scheme
	(read-json "{\"jsonrpc\": \"2.0\", \"method\": \"Player.PlayPause\", \"params\": { \"playerid\": 0 }, \"id\": 1}")

As you know a string in CHICKEN is wrapper in double quotes. JSON also uses double quotes inside its serialized string.
So to not confuse CHICKEN, we have to escape the JSON double quotes. Escaping of characters inside a CHICKEN string is done with a backslash.
If you input this into your interpreter you will get the following output:

	:::scheme
	((jsonrpc . "2.0")(method . "Player.PlayPause")(params (playerid . 0))(id . 1))

Let us take a closer look at this return value as there are some new things hidden here.

We have four pairs, corresponding to the four entities we have in our JSON-RPC spec.

These pairs are also referred to as *cons cells*, the building blocks of a list inside CHICKEN.
You may notice that the members of each cons cell are separated by a dot. This is the true, correct way of writing down a pair.

Let us first break down the cons cells that are required according to the JSON-RPC spec (version, method, id):

	:::scheme
	(jsonrpc . "2.0")

This cons cell describes the JSON-RPC spec version we will use, being 2.0.

	:::scheme
	(method . "Player.PlayPause")

This cons cell describes the method we send to the server, being a string known to the XBMC api.

	:::scheme
	(id . 1)

And here yet another simple cons cell telling which ID this request object will have.

Now let us look at the optional one, the params:

	:::scheme
	(params . (playerid . 0))

This cons cell looks different. In fact, it is a cons cell withing a cons cell.

Here we introduce another part of why list notation is useful: *hierarchy*. You can use lists within lists within lists ...

What the above snippet tells us is that we have a cons cell from which the second element is another cons cell, or I should better say, the *cdr* is another cons cell.

## car, cdr & cons

This gives us a nice little bridge to talk about *car* and *cdr*, two more vital parts of the Scheme programming language.

In our cons cells we used above we have a *car*, the *first* element of the pair and a cdr, the *rest* of the pair.
Car and cdr are two primitive procedures, introduced by John McCarthy himself, that you have available in every Scheme (or every LISP remainder for that matter).

Both procedures have only one parameter being a list and will return one value being either the first element of the list in case of *car* or everything *but* the first element in case of *cdr*.

There is nothing more to it.

Before you can ask the car or cdr from a pair, you have the *have* a pair first. So, if you have two primitive elements, say *strings*, how can we make a pair out of them?

Using *cons*!

Cons is another primitive procedure that *creates* a pair.
This procedure takes *two* arguments and returns the two items *consed* onto each other.

To demonstrate, try this in the REPL:

	:::scheme
	(cons "foo" "faa")

It will result in:

	:::scheme
	("foo" . "faa")

But the elements don't have to be primitive elements, they can also be lists.
That means we can do this:

	:::scheme
	(cons "foo" '("faa" "fii"))

And get this in return:

	:::scheme
	("foo" "faa" "fii")

We just consed the *atom* "foo" onto the list '("faa" "fii").
Not that difficult I would presume!

Well, I haven't told you everything yet.

First, I just threw a new term at you: the *atom*. An atom in CHICKEN is the most basic *s-expression* that you can find and *almost* always *evaluates to itself*.
If you type the string "foo" inside the REPL, it will return "foo" to you. Meaning the string "foo" evaluates to itself.
Strings, numbers, booleans (true and false) all are atoms and all evaluate to themselves.

There are, however, some atoms that *do not* evaluate to themselves. The *symbol* for instance.
The symbol, most of the time, is used as an identifier for a variable, so it evaluates not to its name, but to the values it is referring to.

Second, it is important to know what a cons cell *really* is, okay, we know it is a pair, but what is the deeper meaning of it?
A cons cell is actually a pair of two *pointers*, two references to places inside your computers memory.
The car of the cons cell points to the s-expression you want to have as the first element and the cdr points to another s-expression.
In case you have a pair, this is what actually happens:

	:::scheme
       ----> "foo"
	   |
	("foo" . "bar")             -----> '()
	           |                |
			   ----> ("bar" '())
	                    |
						----> "bar"

The car of the cons cell ("foo" . "bar") points to the atom "foo" in memory.
The cdr points to another list in memory. You don't actually see it, but every (proper) list ends with the empty list '().
So the cdr of this cons cell is actually not "bar" but rather a small list:

	:::scheme
	("bar" '())

With this profound new knowledge, let us take another peek at the CHICKEN s-expression that Medea gave us.
This time I will indent it a little bit differently:

	:::scheme
	(
	  (jsonrpc . "2.0")
	  (method . "Player.PlayPause")
	  (params
	    (playerid . 0)
	  )
	  (id . 1)
	)

This should now look like a piece of cake (nom . nom)!

We have seen how can get from a JSON string to a CHICKEN s-expression, but what about the other way around?
As seen above we can use the following procedure:

	:::scheme
	(write-json datum)

Let us see if we can convert the s-expression we got above back to a JSON string.
This is rather simple:

	:::scheme
	(write-json '((jsonrpc . "2.0")(mathod . "Player.PlayPause")(params (playerid . 0))(id . 1)))

Actually, this is exactly the same s-expression that we got back from Medea!

Well, almost, there is one small detail: we put the whole s-expression inside a *quoted* list.

This is a small example of the *data is code and code is data* statement from earlier.
By putting this s-expression inside another list that has a quote before it, CHICKEN knows that this has to be treated as data.
If you drop the quote, CHICKEN will see it as code and try to evaluate it.

It may not seem as much right now...but this is **AWESOME**, ahum...sorry for that.

Good, we now have a *basic* understanding of how we can use Medea.
We also know what s-expression we have to build in order to be JSON-RPC valid.

What's next?

Now we have to actually *construct* the code that will build this JSON-RPC valid s-expression.
Also, we have to give the user of our egg an easy to use interface for putting in its data.
We don't want them to be bothered with the whole JSON-RPC and Medea thing, that all happens behind curtains.
We want them to be able to call a simple procedure with a few arguments and be done with it.

## Hatching an egg

So I think it is time to actually start to create our egg file! *Finally!*

A basic egg in CHICKEN is nothing more then a directory containing four files:

* *The source code file(s):*
  This will contain the actual code that makes up your egg
* *A setup file:*
  Contains information on how to compile your egg
* *A meta file:*
  Contains information about the license, author, documentation, title, etc. for your egg
* *A release info file:*
  Contains the location of your egg on a code hosting site and the version(s) you release

These last three files only matter for when you are ready to release your egg, stuff we will do in chapter 2 (that rhymes).

For now we can concentrate on building the code for our JSON-RPC egg (the source code file).
Open up your favorite editor and create a file called *json-rpc-client.scm*.

We know that we want to communicate with a server, this implies we need to *talk* with the server over a *connection*. On a OS level, this connection is made by connecting to a *port*, the same as you are now connected to port 80 of the Shisaa server to read this very post. In CHICKEN ports can be represented by, well, *ports*. They carry the same name, but are something different of course. The *ports* in CHICKEN are *objects* that you can write to or read from. It is even possible to make a *connection* with *ports* to a string. You can then wrtie to a string or read from it. In CHICKEN, reading a writing is done from two different ports, the *output* and the *input* port.

Because the *ports* system in CHICKEN is so flexible, we can use it to communicate over a variety of protocols. One of the more common ones is TCP, which is used to communicate with other machines using an *IPv4* or IPv6* address and a *physical* port number. Others include HTTP, WebSockets, ...

Our XBMC server, for example, supports three different protocols to communicate over: TCP, HTTP and WebSockets. Since we are writing a *general* implementation of *JSON-RPC* we cannot make any assumptions as to which protocol the user wants to use. So for now we don't have to worry about which protocol to use, we only need to make sure one is an input port and the other is an output port.

The other important thing to note is that the server we communicate with supports a certain version of the JSON-RPC specification.
When laying a connection with the server, we also want to declare which version of the spec we want to use.

These three things are all we need to setup a connection and they are all we need to *define* our procedure that we will call *json-rpc-server*:

	:::scheme
	(define (json-rpc-server input output version) ... )

There we have it, our first line of code and the first line of our procedure. We defined it, set the name to *json-rpc-server* and gave it three arguments *input*, *output* and *version*. The three dots indicate that this code is not finished yet and thus cannot be copy and pasted into the REPL.

What would be even better is to explicitly set our version to "2.0", because that's the only version we will support in this egg:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0")) ... )

Now "version" is set to "2.0" by default and the user can omit it when calling this procedure.

Next we have to write the *body* (the part of the procedure that holds the actual code) of this procedure, because now it is empty, and thus not quite useful.

The first thing we need in this procedure is to check if the given arguments are valid. And if not valid, give the user some useful feedback.
To check each argument, we can use so called *predicates*. These are procedures that check their argument and simply return true or false.
Build in to CHICKEN are predicates to check for a valid input-port or output-port, so let us use them to check our first two arguments:

	:::scheme
	(input-port? port)

and

	:::scheme
	(output-port? port)

Neat!

For the third parameter, however, we will have to do a little bit more work. Because "version" is something specific to our egg, we will have to make our own predicate.

To do this, simply start to declare a new procedure which we will call *is-valid-version?*.
It is common to put a question mark behind the name of a predicate to distinguish them from "other" procedures. In contrast with many other languages, in CHICKEN you can use almost *any* character inside a variable name. So a question mark is perfectly fine! Just don't use characters like *(*, *)* or *'* or you may confuse CHICKEN into thinking it is the start or end of a list. You can, of course, *escape* characters inside variable names, but you do not have to worry about that now.

From reading the JSON-RPC spec we know that *version* has to be a string containing exactly "2.0"...so that is what we have to check for:

	:::scheme
	(define (is-valid-version? version)
      (string=? version "2.0"))

That's it, we define *is-valid-version?* with only one argument *version*.
Next we check with *another predicate* called *string=?* if the given argument "version" is equal to "2.0".

Instead of using *string=?* we could also have used *equal?*, but *equal?* is a generic predicate then can not only check strings, but many other types as well.
But because we *know* that *version* is a string  we can be more specific and only preform a *string comparison* here.
By doing that, the compiler (who will eventual compile our egg) also has to do less checks and can make some smart shortcuts.

Okay, so now we have three predicates we can use for our three arguments. But *how* do we use them?

Well, put them inside your procedure!

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
    (cond ((input-port? input))
	       ((output-port? output))
		   ((is-valid-version? version))) ... )

The above snippet introduces a new keyword *cond*. You may already guessed it, it's short for *condition*.
And as you can see, we have three conditions. All three have to return *true* or *#t* for the *cond* to continue to the next piece of code.

This is already quite okay, but if one of the predicates is *false* or *#f*, what happens?
Well, the procedure is stopped, without any feedback to the user.

That is not very nice! We have to be as useable as possible, so we have to tell the user *why* the code cannot continue.
To do this, we must provide decent *exception handling*.

### What is exception handling?

In the world of software, *exception handling* is a way to respond to a situation that was not expected.
By good usage of these you can prevent your program from a "crash and burn" situation and you can give helpful feedback to your user.

*Always provide your code with decent exception handing!*

In CHICKEN [you have various ways](http://wiki.call-cc.org/man/4/Exceptions "CHICKEN Exceptions WIKI page") of doing so, one of them being *signal*.

With *signal* you can raise a *continuable* exception. This means that the program does not stop, but simply alerts the user and waits for further input.
We can use *signal* inside a procedure that we make, a custom exception handler that we will construct specifically for setting up a correct JSON-RPC connection.

Let me give you the full code of such an exception handler:

	:::scheme
	(define (server-setup-arguments-error type expected given)
	  (signal
	    (make-property-condition
	      'exn 'message
		    (sprintf "Cannot setup connection, the ~S is invalid. Excepted ~A type but got ~A."
			  type expected given))))

Let us chew this one, piece by piece.

As you are already familiar with, we define a new procedure that I decided to call *server-setup-arguments-error*, that accepts three arguments.
This procedure calls *signal* (to make it a continuable exception) which accepts one parameter, in our case *make-property-condition*.

*Make-property-condition* is a CHICKEN specific procedure that belongs to CHICKEN's [exception handling system](http://wiki.call-cc.org/man/4/Exceptions "CHIKEN WIKI entry for exception handling").
It accepts an exception kind *'exn* and key/value pairs, in our case one pair *'message* and *sprintf*.

The *'message* we construct using the procedure *sprintf*. The latter may look familiar from other programming languages.
With *sprintf* you can construct a string with placeholders, and then fill in the placeholders later.
In our case, we fill in these placeholders with the arguments we give to our custom exception handler.

To be able to ue *sprintf* we have to load in the *extras* library. To load in libraries or modules, we can use the *use* procedure as we have seen when loading in Medea on the REPL.
Put this in the top of your file so that the procedures of *extras* always get loaded in:

	:::scheme
	(use extras)

The placeholders themselves are documented in on the [CHICKEN wiki](http://wiki.call-cc.org/man/4/Unit%20extras#sprintf "Sprintf WIKI entry").
We use *~S* and *~A* which are stand for *writing* or *displaying* the next argument.

What is the difference between *writing* and *displaying*?

A very good question! And also an essential question. *Writing* and *displaying* are specified in the *RnRS* Scheme standards and differ in the way they handle outputting the argument they represent. If you *write* an argument you will put the quoted and escaped representation of, say, a string. With *display* you will put the *unquoted and unescaped* representation:

	:::scheme
	(write "foo (bar)")

Will give you the same, quoted and escaped output:

	:::scheme
	"foo (bar)"

Where as

	:::scheme
	(display "foo (bar)")

Will give you a an unquoted and unescaped output:

	:::scheme
	foo (bar)

In our case we wish to display the given and the excepted arguments as there unquoted and unescaped representation.

So now that we have a basic but decent error handling concept, let us see how we can use it if our above predicates turn out to be false.
Let us take a look at the first line of the predicates:

	:::scheme
	(cond ((input-port? input)))

If we simply put our exception handler there, it would look like this:

	:::scheme
	(cond ((input-port? input) (server-setup-arguments-error "input port" "input-port" (get-type input))))

But now something is not quite correct. If you look closely, the above snipped says: If the give input is indeed a valid input-port then trigger the exception handler.
The is not what we want, we want it to trigger when it is *not* a valid input-port. How? Simple, we use *not*:

	:::scheme
	(cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input))))

Now we have "inverted" the condition. If (input-port?) is *#f*, then trigger the exception handler. If it is *#t*, just continue to the next line.

If we implement this into all of our predicates, our code would look like this:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
	  (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	         ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	         ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))) ... )

We have three predicate checks (one for every argument), with three exception handlers of the same kind that we call with slightly different arguments depending on the situation.

Among the three arguments that we give to our custom exception handler is hidden another procedure (get-type) that gets one argument.

Why is that?

Well, as we said before, we want to be as helpful to our users as we can, that is one of the main reasons that we want to use decent exception handling.
But I wanted to go a small step further by making a message that gives a little bit more information about the arguments the user entered.

In this case, I want to tell the user which data type I expected and which data type they gave the procedure.
For this I created so called *helper function*, *helper procedure* or *helper* that checks which data type the given argument is and returns some information that we can print it in our message.

As you can see, this *helper* is called "get-type" and accepts one argument being the argument that the user gave us.
The helper consists of mere predicates for the most common data types we have. It looks like this:

	:::scheme
	(define (get-type x)
    (cond ((input-port? x) "an input port")
	       ((output-port? x) "an output port")
	       ((number? x) "a mumber")
           ((pair? x) "a pair")
           ((string? x) "a string")
           ((list? x) "a list")
	       ((vector? x) "a vector")
	       ((boolean? x) "a boolean")
		   (else
		     ("something unknown"))))

By now, this sort of thing should not look scary to you at all.

What happens here is we have a *cond* expression which checks a series of conditions.
If one of the given conditions return true it returns with a string, if none of them are true, the final expression, *else*, will be returned being *something unknown*.

To illustrate in further detail how this whole chain of procedures and predicates work, let us assume we call our procedure with an erroneous argument:

	:::scheme
	(define xbmc (json-rpc-server valid-input "foobar"))

We call our *json-rpc-server* procedure with two arguments (version is implicit). Our input-port argument is perfect, our output-port is not so.
What happens in our chain? Let us see step by step:

First the procedure enters the predicate checks. The first one we encounter is our input check:

	:::scheme
	((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))

Since our input-port is correct, this predicate returns true and the *cond* statement goes to the next line:

	:::scheme
	((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))	

Now the predicate returns *#f*, the string "foobar" is not a valid output-port, and thus the *not* statement is executed.
The *not* statement has our custom exception handler as its second argument, so that one is called.

While calling this *server-setup-arguments-error* handler, its arguments are parsed.
The first two are simple strings, but the third is our *helper* "get-type" with the given output-port as an argument, being the string "foobar" here.

Before we can enter the *body* of our exception handler procedure, we first have to find out what the return value of "get-type" is.
When we call "get-type" here, it will look like this:

	:::scheme
	(get-type "foobar")

Since "foobar" is a string, the following predicate will return *#t*:

	:::scheme
	((string? x) "a string")

The argument "x" being our string "foobar". this will make the procedure end and return "a string" to its *caller*.
Our *caller* is the exception handler that is waiting for this return value so it can know its third argument and can continue to its *body*.
We now know this third argument and our call now looks like this:

	:::scheme
	((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" "a string"))

So now the exception handler can finally execute and throw an exception with a custom message, constructed with the arguments we gave it.
The messages returned will look like this:

	:::bash
	"Cannot setup connection, the output-port is invalid. Excepted output-port type but got a string."

Okay, we now have a decent start, we check our user input and respond to it in a correct manner.
It's now time to build the rest of our procedure.

If all the predicates pass, what should happen next?

Well, we want this procedure to return *another* procedure that the user can use to actually send the commands.

We have seen how we can call our procedure:

	:::scheme
	(define xbmc (json-rpc-server valid-input-port valid-output-port))

Now we have defined *xbmc*. When we now call *xbmc*, this will actually be a new procedure waiting for its method and optional params.
Like we said before, when defining *json-rpc-server*, in this case *xbmc*, we define the connection.
Later, we can use this connection to send over the commands by calling this new procedure.

You can see it as a keyboard in a box. By correctly calling our *json-rpc-procedure* we have unlocked the box.
In that box we find a keyboard that is now linked to our server and that we can use to type in our commands (by calling it with correct arguments).

Not making sense yet? Don't worry!

Let us first continue writing our code.

How do we continue after our predicates return *#t*? When using a *cond* you can do this by using the *else* statement.
In our code this would look like so:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
    (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	       ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	       ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	       (else ...

After this *else* we can now put the code that will be executed when all the arguments where correct.
Again, we want *json-rpc-server* to return another procedure that the user can call to actually send something over the wire.
How to we make a procedure return another procedure?

With *lambda*!

### What is this Lambda?

*Lambda* or *Lambda Calculus* is a system in computer programming that allows you to write functions or procedures without naming them.. that's all you need to know for now.

Actually, we have been using lambda's the whole time, I just did not tell you. Every time you declare a new procedure in CHICKEN, you are using a lambda.
When *defining* a new procedure, we wrote this:

	:::scheme
	(define (foo arguments))

But this is actually a shorthand for this:

	:::scheme
	(define foo
		(lambda (arguments)))

But because this is quite a cramp to type every time, we can use the shorthand notation.

Back to our egg, we want our procedure to return a procedure.
Here we cannot use a shorthand, we have to tell it to return a *nameless* procedure or lambda.
Implementing this, it would look something like this:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
      (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	         ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	         ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	         (else
	           (lambda (method . params)))) ... )

Here is our entire procedure so far, returning a nameless procedure (lambda) if all the given arguments where correct.
And now our story starts over again, well, partially anyway. Since we now enter a new procedure with its own, new arguments, we have to do the same thing as before.

What is that?

Why, check the user input and throw exceptions if needed of course!

But that will be food for chapter 2, for this is already enough to chew on in one chapter.

Congrats if you have made it this far and if you understood my blabberings.
Before I leave you hanging for the next chapter, let us take a quick review of what we have seen:

* We have seen that Scheme is not so strange as many people think it is
* We have seen how we could get CHICKEN installed and running on our system
* We have briefly touched on compiling
* We have seen an outline of what we want to build in these three chapters
* We have seen an egg called Medea for converting CHICKEN data types into JSON data types and vice versa
* We have seen how to use this egg to convert both ways
* By doing so we have seen what s-expressions, atoms, cons cells, car, cdr and cons are
* We have seen the skeleton of an egg
* We have laid the basis of our egg and looked at exception handling, predicates and lambdas

That's it for know folks. Relax and clear your mind!
If you did not understand something I suggest you go and read it again until the mental light bulb flips on.

The next chapter will be online soon!

And as always...thanks for reading!
