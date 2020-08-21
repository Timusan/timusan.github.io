<!--
.. title: The Scheme programming language AKA The CHICKEN hens nest - Part 3
.. slug: chicken-scheme-3
.. date: 2013/09/21 10:00:00
.. tags: chicken, scheme, json rpc
-->

## Chapter 3 - Wrapping the egg

And here we arrive at the final stage of our egg development.

If you did not yet do so, please go and read [chapter 1](http://shisaa.jp/postset/chicken-scheme-1.html "Chapter 1 of the CHICKEN series.") and [chapter 2](http://shisaa.jp/postset/chicken-scheme-2.html "Chapter 2 of the CHICKEN series.") before you endeavor on this final hop to a published egg in CHICKEN.

Let us find out what we will be dealing with:

* Explain a little bit about *why the hell we need to write tests for our code*
* Look at how we can actually write basic tests in CHICKEN
* While writing our tests, look at some new items like "let" and "apply"
* Create our setup file to automatically setup our egg when people install it
* Create the needed meta file for the CHICKEN egg system
* Create the release-info file for CHICKEN's code host independent deployment system 
* Quickly compile and install our egg
* Submit the egg to CHICKEN
* Write the egg's documentation

You have got your battle axe ready? Then by all means...dive in!

### Why tests?

First, let me explain to you the importance of writing tests for your code:

**IT IS FRACKING IMPORTANT!**

Ahum...sorry, got carried away there for a bit...let me rephrase that:

Writing tests is rather smart to do.

Why?

Well, it is actually a cultural thing inside a programmers mind or inside a community of programmers. Some (many) programmers find writing tests a cumbersome task, time consuming.

Their train of thought usually runs past *Lazy Ville* and stops at *I'm Freaking Awesome - Don't Need Tests* station.

Phrases you often hear are: "My code works, screw your tests!" or "I can test my code better then a computer can" or "I wanted to write tests, but then my cat sat on my keyboard, switched on YouTube and watched other cats do silly things. It was awesome so I decided to not write tests tonight...I will do that first thing in the morning <small>after washing the car, letting out my cat, putting out the garbage, reading my awesome Facebook page, ...</small>".

The problem, however, is that we are all stupid dumb ass bags of watery flesh. Thinking that we never make mistakes and therefor do not need sufficient testing of our work is downright ignorant and even arrogant. We *all* suck at programming. Some of us less, some of us more. We humans have a tendency of slacking. We let our brains trick ourselves into thinking we always have the whole picture in our minds and therefor know what we are doing.

In other engineering professions where folks build bridges or airplanes, there is a strong sense of not making the same mistake twice (and thus spare a life or two). People learn from mistakes made in the past and implement a huge amount of time and money into testing their work before releasing it to the public. Somehow, in the programming world we dwell in, we tend to neglect all of these values as if we are some kind of super humans. We are not.

Good programmers implement the same engineering values as our neighboring colleagues: they learn from others mistakes and they test their work.

Got it?

Good...then let us write some tests!

### Testing in CHICKEN

Because testing is so vital for every serious project, the CHICKEN community has put a lot of effort into making this as trivial as possible to setup.
Especially the work of [Alex Shinn](http://wiki.call-cc.org/users/alex-shinn "CHICKEN User page of Alex Shinn.") and  [Mario Domenech Goulart](http://wiki.call-cc.org/users/mario-domenech-goulart "CHICKEN User page of Mario Domenech Goulart.") made our CHICKEN testing lives a lot easier.

Alex created an egg called "*Test*" which gives us a bunch of handy procedures to easily write tests for our code. Mario has written an *immense* egg neatly called "*Salmonella*" that you can use to automatically test your egg. Once you submit your egg to CHICKEN, the server will run Salmonella automatically every night on all submitted eggs. It will generate a report that you can check at any time to see if you egg fails its tests.

For my JSON-RPC egg I decided to write two different kind of tests:

* A normal test where we tell CHICKEN which result we except with the given code
* An error test where we explicitly test if something throws an error

Let the writing commence!

All of your tests go into one file called *run.scm* and for Salmonella to find your tests that file has to be placed inside a directory of your egg called *tests*.

In the *run.scm* the first thing you have to put is a line that will actually load the test suite for you:

	:::scheme
	(use test)

Next we need to load in our egg itself, since this is an external test file. Once your egg is compiled you can load it using the *use* procedure like so:

	:::scheme
	(use json-rpc-client)

We can, of course, pull these two lines together to form:

	:::scheme
	(use test json-rpc-client)

This will not yet work since we still have to compile and install our egg. First, let us write the test file.

When testing it is a good idea to divide your tests into groups so that the reports that will roll out later can group your tests to make everything more readable.
For this we can use *test-group*:

	:::scheme
	(test-group "A name for your group" (test 1) (test 2) ... )

You define a name for the group and then list all the tests you would like to perform.
In my case, I would like to test if the JSON-RPC call I do gets formatted to correct JSON-RPC. Therefore I named my first group "JSON-RPC string output checks".
We will first concentrate on writing some tests and later wrap this group around them.

A normal test is very easy to setup and uses easy to read syntax:

	:::scheme
	(test "Description of the test" "The result you want to see" (procedure to test))

We start a test by using the "test" procedure and we give it a description.
Next we set the result that we *expect* to see if our code is correct, this can be a string, a list, anything that could be returned.
Finally we call the to be tested procedure itself.

We want to test that the final JSON-RPC string, that will be sent to the server, is correctly formatted.
Because during testing, we generally do not have access to a real JSON-RPC server to communicate with and thus cannot setup any real world ports to communicate, we need to setup a different kind of port.
These kind of ports are called *string ports*:

* An *input* string port holds a string that you define
* An *output* string port captures strings written to it that you can read out

Once we have these string ports setup we can use it to catch the string that normally would be send to a JSON-RPC server and compare it with a string we defined.
Before testing, we have to define the ports and setup the JSON-RPC connection, in our case called "xbmc":

	:::scheme
	(define input (open-input-string "some-string"))
	(define output (open-output-string))
	(define xbmc (json-rpc-server input output "2.0"))

Now we can throw something at the simulated server and check how it comes out the other end:

	:::scheme
	(xbmc "Player.PlayPause")
	(test "Call with only a method" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"id\":\"1\"}" (get-output-string output))

We call "xbmc" with only a method. Then we call "test", give it a description, give the string we are expecting and finally read out the output port that our xbmc call used with the build in "get-output-string" procedure.

While this test works fine, the whole is not very flexible.
Let me illustrate this by showing you what happens when we create a second test in our "run.scm" file:

	:::scheme
	(xbmc "Player.PlayPause" playerid: 0)
	(test "Call with a method and a one dimensional params" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"params\":{\"playerid\":0},\"id\":\"1\"}" (get-output-string output))

With this test, we not only check the method, but we also give a param and check how that turns out. This is also a perfectly legal and sensible test.

The problem here is that the *output* port we defined above will contain both the output of the first *and* the second test. It is a characteristic of the string output port that it will accumulate the strings that it receives. So when we read from that port we will again receive the result of the first test followed by the result of the second test, which is the one we are actually interested in.

This, of course, means that our second test will fail, because the string comparison will be #f (false).

To solve this problem, we need to create our own little test procedure specifically tailored for our JSON-RPC server. This procedure will just be a small wrapper around the normal "test" procedure, but will take care of the accumulating output port problem described above. Let me present the code to you:

	:::scheme
	(define (test-server description expected method . params)
		(let ((output (open-output-string)))
			(apply (json-rpc-server input output "2.0") method params)
			(test
				message
				expected
				(get-output-string output))))

Okay, there are some new things in here, so let us break this down line by line.

The first line:

	:::scheme
	(define (test-server description expected method . params)

This is nothing new, we define a new procedure called "test-server" which takes three mandatory and one optional argument.
The first argument will be the description we print in our test report, the second argument is the "datum" we except.
The third and fourth arguments are the familiar method and optional params we use in our testing.

Next line:

	:::scheme
	(let ((output (open-output-string))) ...

Here we have a new "thing" in sight: *let*.

"Let" is actually not a procedure, but a "special form" that is quite commonly used when programming in CHICKEN and is needed for *local scoping* of your variables.
The difference between procedures and special forms is (way) beyond the scope of this chapter, just know there is a difference between the two.

#### Local scoping?

In almost all programming languages you have a concept called *"scoping"* and it is nothing more then the name suggests. It is the process of keeping parts of your code (variables, procedures, ...)  "hidden" from other parts of your code. When you scope variables, you make them available to only a certain part of your program. Other parts of your code do not even know that those variables exists and thus cannot read or overwrite them.

In CHICKEN, and most other Scheme languages, "let" will do that job for you. How does "let" work? Simple:

	(let ((foo "a local string")
		  (bar "another local string"))
	     (cons foo bar))

In a "let" you have to define your local variables first, in our case "foo" and "bar". These definitions form the first argument.
The second argument to "let" is the procedure body; the code that will be executed using your locally stored variables.
Code that resides outside of the "let" never knows the existence of these local variables "foo" and "bar".

In our case, "let" can help us by making the output port a local one so that every time the "let" is called, a new output port will be created and thus will only contain one string at a time.
 
The next line is the start of the body of the "let":

	:::scheme
	(apply (json-rpc-server input output "2.0") method params)

Here we encounter yet another new player in town: *"apply"*. This procedure takes an "infinite" amount of arguments; the first always being a procedure and the rest being arguments that the procedure we be applied upon. A very important thing to note about the arguments is that the last argument *has* to be a list, otherwise "apply" does not work.

"Apply" will cons the arguments you put in between its first and last argument onto the last argument, before calling the given procedure with those arguments. Let me demonstrate:

	:::scheme
	(apply some-procedure '(1 2 3))

Now "apply" simply has two arguments, a fictional procedure called "some-procedure" and a list of three arguments '(1 2 3). It is allowed to put additional arguments *between* the two given here:

	:::scheme
	(apply some-procedure 4 5 "string" '(1 2 3))

Now three extra arguments will be placed in between the original procedure and the original, last argument '(1 2 3). "Apply" will first cons every argument onto the last list given:

	:::scheme
	(apply some-procedure '(4 5 "string" 1 2 3))

Then it will take that list and call the procedure with every item, as if it where the actual arguments to that function.

In our case, we have the procedure "json-rpc-server" with its own arguments, one of which is the locally scoped output port. Because this procedure will return a lambda expecting a method and optional params, we can use "apply" to give these arguments to this returned lambda. "Apply" will in turn return the result of the lambda with the method and params applied.

Next we have the rest of our "let" body:

	:::scheme
	(test
		description
		expected
		(get-output-string output))

Here we simply do what we did before: we call the "test" procedure, give it a description, tell it what we expect to get and finally read out the local output string port to compare with.

Let us use this newly created procedure in our test:

	:::scheme
	(test-server "Call with only a method" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"id\":\"1\"}" "Player.PlayPause")

Now we have a piece of code that is a little bit more readable and tailored to our JSON-RPC egg:

* We call the test-server procedure
* First argument is just a description of what we will be testing
* Second is the string we expect to get
* Last is, in this case, the method we want to test our "json-rpc-server" procedure with

One more thing to note is the escaping that you may have noticed in the string we expect. Because a double quote " means something in CHICKEN (it wraps a string) we need to escape it inside the string by using a backslash. The JSON-RPC valid JSON string we use here normally looks like this:

	:::json
	{"jsonrpc":"2.0","method":"Player.PlayPause","id":"1"}

And with the double quotes escaped in a CHICKEN string becomes this:

	:::scheme
	"{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"id\":\"1\"}"

Now we have a valid test for the case where we call our procedure with only a method. Let us now write one with a method and a single param:

	:::scheme
	(test-server "Call with a method and a one dimensional params" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"params\":{\"playerid\":0},\"id\":\"1\"}" "Player.PlayPause" playerid: 0)

Nothing strange happening here. We give it a different description of course, and we expect a different string, including params this time. And at the end we, of course, include a param in there.

Let us now put these two tests in a group, like we said we would do a moment ago:

	:::scheme
	(test-group "JSON-RPC string output checks"
	    (test-server "Call with only a method" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"id\":\"1\"}" "Player.PlayPause")
	    (test-server "Call with a method and a one dimensional params" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"params\":{\"playerid\":0},\"id\":\"1\"}" "Player.PlayPause" playerid: 0))

There, this does not look scary, right? We have a test group called "JSON-RPC string output checks" containing the two tests we just have written.

You know what? That is all there is to basic testing. We have just written a few simple tests to verify the whole purpose of the JSON-RPC client side egg.

The full code that we have in our "run.scm" up until now looks like this:

	:::scheme
	(use test json-rpc-client)

	(define input (open-input-string "just-a-test-string"))
	(define output (open-output-string))
	(define xbmc (json-rpc-server input output "2.0"))

	(define (test-server description expected method . params)
		(let ((output (open-output-string)))
			(apply (json-rpc-server input output "2.0") method params)
			(test
				description
				expected
				(get-output-string output))))

	(test-group "JSON-RPC string output checks"
	    (test-server "Call with only a method" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"id\":\"1\"}" "Player.PlayPause")
	    (test-server "Call with a method and a one dimensional params" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"params\":{\"playerid\":0},\"id\":\"1\"}" "Player.PlayPause" playerid: 0))

A few things to note about the above snippet:

* We use "use" in the beginning, but this does not work yet until we actually compile and install our egg.
* We define a *global* input and output port plus a JSON-RPC connection called xbmc, we will need this later on.
* The input port we defined globally is used in our "test-server" procedure, but the output port is locally scoped in that same procedure.

The next kind of tests we want to write are error tests. This sort of test does exactly what you would expect, test if something gives an error.
In our case we need these kind of tests to see if our error handlers work correctly.

The first error handler we wrote is the one for our main "json-rpc-server" procedure. So to write an error test for this, we simply have to do an erroneous call.
If we recall our "json-rpc-server" procedure we need three things to correctly setup the connection:

* A valid input port
* A valid output port
* A correct version number (a string that equals "2.0")

If we would like to test if it fails when we use a faulty input port, we would write the test like so:

	:::scheme
	(test-error "Non port call on input" (json-rpc-server "input" output "2.0"))

The syntax looks quite familiar. We call the procedure "test-error" instead of "test" and give it a description. Then we simply call our procedure with some kind of error in it.
In this case we give the string *"input"* instead of the defined variable *input*. This string is not a valid input port of course, so our error handler fires and we get an error.
Getting an error in this case means that the test will pass!

In my test I include some more error checks:

	:::scheme
	(test-error "Non port call on output" (json-rpc-server input "output" "2.0"))
	(test-error "Non correct version number call" (json-rpc-server input output "3.0")))

And of course, this can become a group, giving you this code:

	:::scheme
	(test-group "Non-port or non-version calls"
	    (test-error "Non port call on input" (json-rpc-server "input" output "2.0"))
	    (test-error "Non port call on output" (json-rpc-server input "output" "2.0"))
	    (test-error "Non correct version number call" (json-rpc-server input output "3.0")))

A tricky thing to note about error tests is that *any* kind of failure will make the test pass. This can be very misleading because even real errors in your code can cause a failure to occur and the test to pass. To more correctly setup these error tests, I should also check if the error I get back is of the custom type I defined specifically for the JSON-RPC egg. This way I know for sure that it is *my* custom error handler that raises the situation, and not something else. But that kind of setup is beyond the scope of this chapter. For now just remember to be careful when writing error tests.

A very important last step in your "run.scm" file is to end the file with the following code:

	:::scheme
	(test-exit)

If you omit this line the automated tests on the server will fail, so it is vital you end your file with this one.

Good, we finished writing our tests!

To recapitulate, here is the total code we have written so far:

	:::scheme
	(use test json-rpc-client)

	(define input (open-input-string "some-string"))
	(define output (open-output-string))
	(define xbmc (json-rpc-server input output "2.0"))

	(define (test-server description expected method . params)
		(let ((output (open-output-string)))
			(apply (json-rpc-server input output "2.0") method params)
			(test
				description
				expected
				(get-output-string output))))

	(test-group "JSON-RPC string output checks"
	    (test-server "Call with only a method" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"id\":\"1\"}" "Player.PlayPause")
	    (test-server "Call with a method and a one dimensional params" "{\"jsonrpc\":\"2.0\",\"method\":\"Player.PlayPause\",\"params\":{\"playerid\":0},\"id\":\"1\"}" "Player.PlayPause" playerid: 0))
 
	(test-group "Non-port or non-version calls"
	    (test-error "Non port call on input" (json-rpc-server "input" output "2.0"))
	    (test-error "Non port call on output" (json-rpc-server input "output" "2.0"))
	    (test-error "Non correct version number call" (json-rpc-server input output "3.0")))

	(test-exit)

But before we can run our tests, we first have to build some extra files that in total will define our egg and make it able to be compiled.

### The setup file

file: *"json-rpc.setup"*

This file mainly contains information for the compiler and for the CHICKEN install system. Check the [wiki page](http://wiki.call-cc.org/eggs%20tutorial#the-setup-file "Setup file wiki page on CHICKEN.") for more information.

The JSON-RPC eggs setup file looks like this:

	:::scheme
	(compile -s -O2 -d1 json-rpc-client.scm -j json-rpc-client)
	(compile -s json-rpc-client.import.scm -O2 -d0)

	(install-extension
		'json-rpc
		'("json-rpc-client.so" "json-rpc-client.import.so")
		'((version "0.1.4")
		  (documentation "json-rpc.html")))

The first two lines are the lines the compiler will use to compile your "scm" files into actual raw C code.
The top line contains the CHICKEN file we have been working on in chapter one and two. The flags that are set here are a sensible default, what they mean is beyond the scope of these posts. You can, in most cases, just use these flags as is.

The second line contains a CHICKEN file called "json-rpc-client.import.scm" which we did not create but which will be automatically created for you.
This file will contain information for the CHICKEN module system and also needs to be compiled to C.

The rest of the file is occupied by a list containing some meta data about your egg. Let me break this down for you:

* 'json-rpc - This is the actual name of your egg
* '("json-rpc-client.so" "json-rpc-client.import.so") - Will be the compiled files, the two CHICKEN file we mentioned above, but just with the extension "so" instead of "scm"
* '(version "0.1.4") - The version that you wish to compile. Every time you update the version you should *change* this here as well
* '(documentation "json-rpc.html") - The place where CHICKEN can find your documentation. I have placed my documentation on the CHICKEN Wiki, which is the standard. If you place it on the wiki, all you need to do is to put the name of you egg (in this case "json-rpc") with the extension ".html" in there. Next it is important to also actually create that page on the wiki. We will do this in a bit.

### The meta file

file: *"json-rpc.meta"*

The meta file contains all kinds of information related to your egg. This information will later be displayed on the CHICKEN egg pages. Check the [wiki page](http://wiki.call-cc.org/eggs%20tutorial#the-meta-file "Meta file wiki page on CHICKEN.") for more information.
In the case of my JSON-RPC egg I input the following information:

	:::scheme
	((egg "json-rpc.egg")
	 (synopsis "JSON RPC client/server implementation")
	 (category web)
	 (needs medea)
	 (test-depends test)
	 (doc-from-wiki)
	 (license "BSD")
	 (author "Tim van der Linden"))

Let us go over them:

* Give the eggs name, together with the extension ".egg"
* Write a short synopsis about the function of your egg
* Say which category the eggs belongs to. The different categories are listed [here](http://wiki.call-cc.org/eggs%20tutorial#the-setup-file "Egg categories wiki page on CHICKEN.")
* Tell CHICKEN which *non-core* dependencies your egg has. In the case of the JSON-RPC egg, we have "srfi-1", "medea" and "extras", but only "medea" is a non-core dependency.
* Give the type of documentation you have. If your documentation resides on the CHICKEN Wiki, you need to put this line in
* Tell about the license you have included in your ".scm" files
* Tell them who the author is

That is all that goes in the meta file, save and close it.

### The release-info file

file: *"release-info"*

CHICKEN has the unique ability to be totally code host independent.

You can put your code on CHICKEN's own SVN repositories or on your favorite code hosting site and the CHICKEN egg server will pull all information from there.
The "release-info" file contains information about where your code is hosted and which version numbers you have available.
Even when your third party code host is down, CHICKEN still has a local copy of the latest available version of your egg, so people installing your egg can still continue their development.

Depending on which code hosting site you use there are different settings you ave to configure. There is an [extensive page](http://wiki.call-cc.org/releasing-your-egg#creating-a-release-info-file "Release-info file creation on the CHICKEN wiki.") about how to setup your release-info for your code host.

I personally resent the hype around GIT (among other things) and chose Mercurial as my version control system and Bitbucket as my host. So the settings in the release-info file became:

	:::scheme
	(repo hg "https://bitbucket.org/Timusan/{egg-name}")
	(uri targz "https://bitbucket.org/Timusan/{egg-name}/get/{egg-release}.tar.gz")
	(release "0.1")
	(release "0.1.1")

What is means:

* The first line is the url to the eggs main location
* Then we have the url to the gzipped tar files for each egg release I make (using Mercurial tags)
* And finally we have the different versions listed that are released

### Doing a test install

Okay, we now have the correct environment to install our egg via the *chicken-install* command we used in chapter one.
Make sure you are in the main directory of your egg and that you are root. Then simply call "chicken-install" without any arguments:

	:::bash
	$ chicken-install

Or if you do not want to become root, simply use the "-s" switch to temporarily sudo the install:

	:::bash
	$ chicken-install -s

The installer will now look in the current directory for all the needed files and install your egg.

If all went well you will have some compiler output and your egg is compiled and installed in your local CHICKEN ecosystem.
This means you can now run your tests!

You can either run "Salmonella" that will not only run your "run.scm" tests file, but also will do checks on the availability of the documentation among other things.
Since our egg is not published yet, those checks would fail. But by simply running the "run.scm" file by itself, we can run our tests without anything interfering.

So go into your "tests" directory and run your CHICKEN file with "csi":

	:::bash
	$ csi run.scm

CHICKEN will now print a neat little report with the results of our tests. Cool!

### Publishing & Documentation

You now what? You are ready!

The only thing left for you to do now is to first commit your code to your favorite code hosting site, then announce the existence of your egg and finally write the documentation for it.

Before you can finally publish your egg for the world to see, you *have* to write some form of documentation.
Since it is common to write the documentation on the CHICKEN Wiki itself, you will first have to ask for an account so you can properly access the Wiki.

After committing your code, send an email to the [CHICKEN Users](https://lists.nongnu.org/mailman/listinfo/chicken-users "The CHICKEN Users mailing list.") mailing list with the location to your egg so the CHICKEN peeps can add your egg into the system.

Your account details are best mailed to the private address of [Mario Domenech Goulart](http://wiki.call-cc.org/users/mario-domenech-goulart "CHICKEN User page of Mario Domenech Goulart."). Check out [this](http://wiki.call-cc.org/contribute "The CHICKEN egg contribute page.") page to find out his email. Mail him your desired user name and your hashed password. To generate the hash for your password, use the "OpenSSL" program:

	:::bash
	$ openssl passwd -apr1 your-password-here

Once you get confirmation of your account creation, you can start writing.

The way this is usually done is to create a page on the Wiki with the same name as your egg.
For the JSON-RPC egg, this would be:

	:::url
	http://wiki.call-cc.org/eggref/4/json-rpc

Say, for the sake of exampling, that your eggs name is "foo-bar", you would surf to:

	:::url
	http://wiki.call-cc.org/eggref/4/foo-bar

The Wiki will tell you that that page does not exist, but it also gives you the chance to create it.
Create the page and be sure to authenticate with your fresh, new credentials when saving.

To get inspiration on how to write the documentation for your egg, check out other contributors their documentation pages. Make sure that you always include the following items:

* A clear description of your egg
* The name of the author
* The non-core dependencies your egg has
* A link to the external repository where you eggs code is hosted
* Documentation about the procedure the users of your egg can use
* Some real-world examples
* Version history
* A copy of the license you use

### The end

Okay, that is it folks, we have a working, tested and published egg!

I hope this three chapter CHICKEN saga has brought you a little bit closer to CHICKEN or Scheme and that you can gradually find out the power behind this tiny language.

Now go out and create some awesome eggs for the whole community to enjoy! And remember, if you run stuck, have questions or just want a little chat, there is a great CHICKEN community out there ready to help you out. 

And as always...thanks for reading!





