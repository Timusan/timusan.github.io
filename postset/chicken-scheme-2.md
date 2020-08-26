<!--
.. title: The Scheme programming language AKA The CHICKEN hens nest - Part 2
.. slug: chicken-scheme-2
.. date: 2013/09/05 19:00:00
.. tags: chicken, scheme, json rpc
-->

## Chapter 2 - Laying the egg

Welcome to the second installment of this introduction into the CHICKEN programming language.

If you haven't done so, I encourage you to go and read [chapter 1](http://shisaa.be/postset/chicken-scheme-1.html "Chapter 1 of the CHICKEN series.") before you continue.

In the first chapter we talked about the programming language CHICKEN, which is a Scheme implementation. To introduce you to this language I decided to guide you trough making your own egg.
This egg will become a translator to talk to a JSON-RPC server so you could send over commands and invoke remote procedure calls trough JSON right out of CHICKEN.

I left you with the beginning of our main procedure called *json-rpc-server* which would do two things:

* It would setup the connection to the server
* It would then be used to translate the commands into JSON and send them over

Before we dive in, let me give you a quick perspective of what we will be dealing with today:

* Create an exception handler for the arguments of the lambda
* See how we can abstract out positional and named params in the arguments
* Build the actual list that will hold the CHICKEN JSON-RPC request object we wish to send
* While building this, we can see a little bit about proper and improper lists
* Still while building, we can also dive into recursion in CHICKEN for building the params list
* After the request object is build, actually send it over to Medea
* Make sure we export only the needed procedures
* Setting the correct license for our egg
* Take a small peek at what lies ahead in chapter three.

### Let us dive in!

We where busy writing the second part of our procedure. Once the connection was setup we would return a *lambda* or *anonymous procedure* to the user which she or he could then use to send commands with.
This is how we left our procedure:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
      (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	         ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	         ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	         (else
	           (lambda (method . params) ... ))))

As you can see, we started to return a lambda that accepts two arguments: *method* and *params*.

As we have seen in the previous chapter, params are *optional*. We thus want them to be optional too in our CHICKEN implementation.
To give optional arguments to a procedure we can use the *dot notation*: everything left of the dot is required and right of the dot is optional.

And as you remember from the first chapter, we have to help the user when they input wrong information. In this case it means we have to check two arguments. But since both method and params are completely custom to our egg, we will have to build the predicates ourselves and then build a custom exception handler that will inform the user about her or his mistake.

We know that the "method" argument must be a string, so the first predicate is quite simple:

	:::scheme
	(define (is-valid-method? method)
      (string? method))

The "params" argument is a little bit different. Since we are implementing JSON-RPC in CHICKEN, we have the chance of also making some aspects more abstract, more easy to use.
As we have seen in the JSON-RPC spec, params can be *positional*, which in CHICKEN becomes a *vector* or they can be *named* which is an *alist*.
So we could opt for the situation where the user has to input either an alist or a vector as a "params" argument, or we could try and find a more simple way of input.

I chose the latter and opted to let the user either input *symbol arguments* (positional params) or *keyword arguments* (named params):

	:::scheme
	('these 'are 'five 'positional 'params) ;symbols
	(these: are keyword: params) ;keywords

Inside the procedure body which implements the params, both keywords and symbols become lists we can use, just as the example above.
So to make the predicate for checking valid params input the only thing we really have to do is to check if the given params are a list.
Even when we do not give any params, they will be considered *null*. And as we have seen in the previous chapter, *null* in CHICKEN is the same as the *empty list* or *'()*, which, of course, is also a list!

Cool, right?

So our params predicate would look like this:

	:::scheme
	(define (are-valid-params? params)
      (list? params))

Next we have to make a new exception handler. This one we will call *server-setup-data-error* and it will be called if one of the predicates fail or in other words, when the method or the params are invalid:

	:::scheme
	(define (server-setup-data-error type message)
      (signal
        (make-property-condition
          'exn 'message
          (sprintf "Cannot setup connection, the given ~S data is invalid. The ~S ~A"
	      type type message))))

Looks quite the same as the first one we wrote in chapter 1, no?

The only difference is the message we print and the arguments we give to this procedure.
We can now implement the predicates and the exception handling the same way we did before, so this becomes:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
      (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	         ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	         ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	         (else
	           (lambda (method . params)
	             (cond ((not (is-valid-method? method)) (server-setup-data-error "method" "can only be a string."))
		                ((not (are-valid-params? params)) (server-setup-data-error "params" "can only be a vector or an alist."))
						(else ... ))))))

I also added the *else* statement at the end which will carry the code that will be executed when the given arguments are correct.

What's next?

We now have to start looking at actually building the JSON-RPC request object in CHICKEN and sending it to server.
First let us concentrate on the building part:

### Building the request object

Let us assume that we already finished our egg and we define a connection to the "XBMC" box I described in the first chapter, we would call our *json-rpc-server* procedure as follows (assuming we have a valid input-port and output-port):

	:::scheme
	(define xbmc (json-rpc-server input output))

When we now call "xbmc" we will get a new procedure (the lambda we are writing now) that excepts a method and optional params.
Given the example from chapter 1 (Player.PlayPause), let us see how we would call "xbmc" to send this to the server:

	:::scheme
	(xbmc "Player.PlayPause" playerid: 0)

*Player.PlayPause* is the method, which is a string and *playerid: 0* is a *keyword* param where *playerid:* is the keyword and *0* the value of that keyword.
We now have to write the code that will take that and turn it into this:

	:::scheme
	(
	  (jsonrpc . "2.0")
	  (method . "Player.PlayPause")
	  (params
	    (playerid . 0)
	  )
	  (id . 1)
	)

The above code is a list, so we can also say that we want to *build* a list containing the *"Player.PlayPause"* and *playerid: 0*.

In chapter one we have seen that the *jsonrpc* version is always *2.0*.

On top of that, we can always keep the *id* at a fixed number.

The reasoning behind the *id* in a JSON-RPC call is that the server can use the same *id* in its *response* or *error* object it returns so you know the server's feedback belongs to your *request*. But because we only send one request at a time, it is overkill to generate a unique *id* for each *request* we send. In our egg, we will keep this *id* at *1*.

Let us concentrate on building the request object from only the *version* and the *id* for now. The *method* and the *params* will need some further attention later on.

So the first thing we encounter in the *s-expression* above is the *cons cell*:

	:::scheme
	(jsonrpc . "2.0")

The "version" is an argument to the "json-rpc-server procedure", so to create this cons cell we can use *cons*:

	:::scheme
	(cons 'jsonrpc version)

Mind the *'* or *quote* before "jsonrpc". This is needed for otherwise CHICKEN will see this as *code* instead of *data* and try to execute it.
The last cons cell is our "id", the same idea applies here:

	:::scheme
	(cons 'id 1)

Again, mind the *'* in front of "id". We use the *number* 1 here without any quoting because we want CHICKEN to treat this as an actual *number* data type and not as a *symbol* or *string*.

How can we put these two *cons cells* together?

Let us try it with cons and see what happens. Remember that cons only accepts two arguments and *conses* them on to each other. So if we want to put the two cons cells together, it will look something like this:

	:::scheme
	(cons
		(cons 'jsonrpc version)
		(cons 'id 1))

This code will result in:

	:::scheme
	((jsonrpc . version) id . 1)

Hmm, that is not really what we want, is it? It has now become a list containing one *cons cell* and two *atoms* separated by a dot.
Why is this happening and why do we not get the following code?

	:::scheme
	((jsonrpc . version)(id . 1))

Good question, the answer is: You have just created an *improper list*.

Improper list?

To explain this we have to take another look at the primitive *cons* procedure we saw in chapter 1. There we learned that *cons* is used to *cons*truct pairs, just as we did above. And with pairs, we can eventually construct *lists*. But there was also one important detail we saw, the *null*, *'()* or *the empty list*. I showed you that when we look at a list, we are actually looking at cons cells and the last item in the list is actually a cons cell that contains the empty list at the end. You do not *see* the empty list, but it is there.

An improper list, as opposed to a proper list, does *not* end in the empty list. In the example above, we ended with the cons cell:

	:::scheme
	('id . 1)

This is considered wrong, it is even said that an improper list is not a list at all. To turn this into a proper list, we simply have to add the empty list into our consing adventure:

	:::scheme
	(cons
		(cons 'jsonrpc version)
		(cons
			(cons 'id 1) '()))

And this will give as a proper list and thus the result we are after:

	:::scheme
	((jsonrpc . "2.0") (id . 1))

Nice!

There is one annoying thing about building lists this way: its verbose. Luckily for us, CHICKEN has a shorthand for building lists, amazingly called *list*.
The above example would thus be translated to:

	:::scheme
	(list
		(cons 'jsonrpc version)
		(cons 'id 1))

That is a little bit less to type, and maybe even more easy to read. The procedure *list* can take an infinite amount of arguments and construct a proper list for you which means that it will always automatically add the *empty list* at the end.

Let us now take this one step further and include a *method* in our construction. If we would use *cons*, the construction would look like this

	:::scheme
	(cons
		(cons 'jsonrpc version)
		(cons
			(cons 'method "Player.PlayPause")
			(cons
				(cons 'id 1) '())))

And if we used *list*:

	:::scheme
	(list
		(cons 'jsonrpc version)
		(cons 'method "Player.PlayPause")
		(cons 'id 1))

Both approaches give us:

	:::scheme
	((jsonrpc . "2.0") (method . "Player.PlayPause") (id . 1))

You are of course free to choose if you want to use "cons" or "list", the take-home message here is that you know *how* to cons together a proper list and that you understand how "list" will create a *proper list* for you by adding the empty list at the final cons cell.

The next bit we have to deal with are our *optional* params. The fact that they are optional means that we cannot simply add them in our consing goodness. We have to take the process of putting together our final list apart and build in the necessary checks to see if the params are given or not.

One way to do this is to simply go about and check if there are any params present and if they are, just cons them onto the list we already build. While this would work fine, there is still a theoretical problem that could stick its head up. When we would cons the params onto the existing list, the params will end up at the *beginning* of the list. You would get something like this:

	:::scheme
	(
      (params
	    (playerid . 0)
	  )
	  (jsonrpc . "2.0")
	  (method . "Player.PlayPause")
	  (id . 1)
	)

This is actually perfectly fine. The order of the *jsonrpc*, *method*, *params* and *id* is arbitrary. In the final JSON string the order of these keywords does not matter and the server you are sending your string to will perfectly understand your call. So the above CHICKEN code is correct.

However, like I said, there is a theoretical problem. In the JSON-RPC specification they use *exactly* the same order throughout the documentation. The *jsonrpc* version is first, then the *method*, then optionally the *params* and finally the *id*. They do this for clarity, of course, but somebody building a JSON-RPC server could interpreted this as part of the spec and only accept JSON-RPC calls that are formatted in that exact order.

As assuming a particular order would be a wrong interpretation of the JSON-RPC spec, you could ignore this. If somebody would use your egg to communicate to such a faulty JSON-RPC server, you simply could blame the programmer who build the server. You would be perfectly correct. Or not...?

You could also think about it from a slightly different perspective. If 99 percent of the JSON-RPC servers do not care about the order, you could as well provide the same order as the spec demonstrated. This way even the faulty servers would still work and the users of your egg will have one frustration less to worry about.

The latter is the path I choose with my egg, I construct my JSON-RPC call in exactly the same order as demonstrated in the specification.

But by choosing this path, we bring forth another interesting challenge. In the order of the examples, the *params* come not at the end, not at the beginning but in the middle of our list.

Luckily for us we are using CHICKEN and this problem is actually quite trivial. We can simply put an *if* condition around our *param* while putting our string together. Let me demonstrate:

	:::scheme
	(list (cons 'jsonrpc version)
		  (cons 'method "Player.PlayPause")
		  (if (null? params) '() (cons 'params 'theparams))
		  (cons 'id 1))

As you can see, we put a simple *if* procedure around our *params* cons cell. When we do not give any *params* we simply return the empty list. When we do have *params* we return those. Note that in the example above I used *'theparams* to substitute for the actual *params* that we have to build later on.

If you would like to try this example in the REPL, we first have to define *params*, otherwise our *if* procedure will fail stating that it does not know *params*. So first, for the sake of testing, define *params* in the REPL:

	:::scheme
	(define params #t)

We now defined it as *#t* or *true* which means it is *not empty*. If you now type in "params" in the REPL, you will get back *#t*.
Now, let us input our slightly modified statement into the REPL:

	:::scheme
	(list (cons 'jsonrpc 'version)
		  (cons 'method "Player.PlayPause")
		  (if (null? params) '() (cons 'params 'theparams))
		  (cons 'id 1))

Notice that I quoted *version* also, because this is still unknown for the REPL in this stage of development.
If you punch this in, you will get the following output:

	:::scheme
	((jsonrpc . version) (method . "Player.PlayPause") (params . theparams) (id . 1))

Neat! We have four *cons cells*, including our params. Seems to work so far, no?

Okay, now let us try it when *params* are not given.
We first need to redefine *params* to not be *#t* but be *the empty list*:

	:::scheme
	(define params '())

And now run our code again:

	:::scheme
	(list (cons 'jsonrpc 'version)
		  (cons 'method "Player.PlayPause")
		  (if (null? params) '() (cons 'params 'theparams))
		  (cons 'id 1))

The output will now read:

	:::scheme
	((jsonrpc . version) (method . "Player.PlayPause") () (id . 1))

Hmmm, you notice the problem? There is an empty list right in the middle of our list.

This, of course, makes perfect sense, since we told CHICKEN to input the empty list if we had no *params*.
But if we would later go and convert this into a *JSON-RPC* valid call, we would run into trouble.

We have to find a way to *remove* this empty list from our resulting list.

How?

With *remove* of course! *Remove* is a procedure that is delivered by the so-called *SRFI-1* and is in CHIKCKEN core.

What is *SRFI-1* you ask?

*SRFI* stands for *S*cheme *R*equests *f*or *I*mplementation and are well defined Scheme libraries. The reasoning is that Schemers can use these libraries to extend the functionality of their Scheme. Somebody building a Scheme implementation can choose which SRFI's he or she would include or implement in its core. If a library proves to be useful enough it even makes a tiny chance of being included into a new RnRS spec. You can check out [the SRFI website](http://srfi.schemers.org/ "The official SRFI repository.") containing all the available SRFI's.

*SRFI-1*, aka the *List Library*, contains several useful procedures for working with lists. One of these procedures is called *remove*.

*Remove* accepts two arguments, a predicate and a list. Whatever matches the predicate will be, well, removed. So in our case, we want to remove everything that is *null* or *the empty list*.
In the previous chapter we already saw that we have a *null?* predicate. Now we can add the procedure *remove* together with that predicate to get rid of our *empty list*:

	:::scheme
	(remove
		null?
		(list
			(cons 'jsonrpc 'version)
			(cons 'method "Player.PlayPause")
			(if (null? params)
				'()
				(cons 'params 'theparams))
			(cons 'id 1)))

Which will result in:

	:::scheme
	((jsonrpc . version) (method . "Player.PlayPause") (id . 1))

Good! The empty list is now gone!

But before we can use stuff from SRFI-1 in our egg, we will have to *load in* its procedures the same way we loaded in *Extras* in chapter one.
To load in multiple libraries or eggs at once, simply list the names as arguments to *use*:

	:::scheme
	(use extras srfi-1)

Let us take a look at the full code we have now for our *json-rpc-server* procedure:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
      (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	         ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	         ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	         (else
	           (lambda (method . params)(
			    (cond ((not (is-valid-method? method)) (server-setup-data-error "method" "can only be a string."))
					  ((not (are-valid-params? params)) (server-setup-data-error "params" "can only be a vector or an alist."))
				      (else
					    (remove
							null?
					        (list
								(cons 'jsonrpc version)
								(cons 'method method)
								(if (null? params) '() (cons 'params 'theparams))
								(cons 'id "1"))))))))))

That starts to look like a nice little program!

But as you can notice, we still have *'theparams* in there, which needs to be replaced with the actual params the user inputs.
In the beginning of the chapter I explained that the *params* can be either an *alist* or a *vector* so that they can later be converted correctly into JSON.But we also saw that we don't want to bother the user with that. We want the user to input simple keywords or symbols.
This means we have to make a translation from keywords/symbols to alists/vectors.

The first thing we need to do is to check what the user has input for *params*. Are it keywords? Or are it symbols? After we know what the user has input, we can build either an *alist* or a *vector*. This calls for three small procedures, one that checks the input, one that builds an *alist* and one that builds a *vector*.

Let us begin with the input checker. This procedure will check the input and call the appropriate procedure, then it will return that result. So let us call this procedure *build-params*. It will take one argument: the params that the user has input. This little fellow will look something like this:

	:::scheme
	(define (build-params params)
		(if (keyword? (car params)) 
			(build-alist params)
			(list->vector (build-vector params))))

As you can see, we have an *if* procedure that checks the *car* of the *params* to be a *keyword*. The *keyword?* procedure is a build in predicate that we have at our disposal. If the *car* is a keyword we have to build an *alist*. So it calls a custom procedure *build-alist*.

Let us forget the rest for a moment and see how we can put together *build-alist*.
Before we start writing code, let us do it in our minds first.

If the user has input keywords, we can safely assume they will be in the following form:

	:::scheme
	(first: "string" second: "string" third: "string")

And we also know the resulting alist we would like to see:

	:::scheme
	((first . "string") (second . "string") (third . "string"))

What we need to do is take the keywords (first:, second: and third:) and make them the car of each pair, then take the value of the keywords and make them the cdr of each pair.

We have already seen almost every tool we need to do this job of putting together these pairs. If we had, for example, three keywords like in the above example, the code to put this together would look something like this:

	:::scheme
	(cons
		(cons (car params) (car (cdr params)))
		(cons
			(cons (car (cdr (cdr params))) (car (cdr (cdr (cdr params)))))
			(cons
				(cons (car (cdr (cdr (cdr (cdr params)))))(car (cdr (cdr (cdr (cdr (cdr params)))))))
				'())))

Wow...that seems quite verbose, no? Look at all the car, cdr and cons procedures, not to mention the unscalability of such code. If we get, for example, four or five keywords, this gets twice as long! So we have to be a bit more lazy and let CHICKEN do that heavy lifting for us. We need some kind of loop.

### Recursion

A loop in CHICKEN or any other Scheme is not like your average loop in Python or C, in CHICKEN there actually is *no* loop. Instead of a loop there exists an idea that is called *recursion*. This is quite a simple idea where you do something over and over again until a certain condition is met. Let me demonstrate:

Say for example, we have a list with three numbers:

	:::scheme
	'(1 2 3)

And we want to do the quite useless exercises of adding one to each of the numbers.

We could do this by writing a tiny procedure that takes the first number, adds one to it, go over to the next number and do this over and over until there are no numbers left.

In code, this tiny *recursive* procedure looks like this:

	:::scheme
	(define (addone list)
		(if (null? list)
			'()
			(cons (+ (car list) 1) (addone (cdr list)))))

The + sign we use here is a build-in procedure that adds up numbers, just so you know. Remember, in CHICKEN you can use almost any character as a variable or procedure name.

Then we call this newly made procedure:

	:::scheme
	(addone '(1 2 3))

And we get back:

	:::scheme
	(2 3 4)

You have just seen recursion in action!

To fully understand what just happened we have to dig in a little bit deeper and take a look at how the *recursion* is happening behind the curtains.

We created the procedure called "addone" that calls *itself* until the so-called *base case* is reached. The base case here being:

	:::scheme
	(if (null? list))

An important detail to know is that every time the procedure calls itself it will be put on a waiting list and wait for the newly called procedure to finish and give back its result.
When the base case is reached, the final procedure call returns its result (in this case the *empty list*) and this empty list will be given to the second last procedure which can cons its result on to that.
This second last procedure then finishes and gives its result to the third last procedure that was waiting. This continues until the first, original procedure gets its result and returns this to you (or the procedure that called it in the first place).

You can think of it as having a big box right in front of you, you open the box only to find another box, you open that one and you find yet another box.
This goes on until there are no boxes in boxes left (the base case is reached). But before you can close the big, original box, you will have to close every box, starting with the most smallest inner box working your way back to the big, original box.

Let us look at each step of the recursion for our "addone" procedure.

*1* - We call "addone" for the first time, the argument is not null, so the base case is not reached. This means we continue with the final cons line.

	:::scheme
	(cons (+ (car list) 1) (addone (cdr list)))

which translates to

	:::scheme
	(cons (+ 1 1) (addone '(2 3)))

You can notice that the "addone" procedure is now called with the argument '(2 3).

*2* - The base case is still not reached, because '(2 3) is not null, we continue again with the last line.

	:::scheme
	(cons (+ (car list) 1) (addone (cdr list)))

which translates to

	:::scheme
	(cons (+ 2 1) (addone '(3)))

The procedure "addone" is now called again, with the '(3) as the argument.

*3* - The base case is still not reached, because '(3) is not null, we continue again with the last line.

	:::scheme
	(cons (+ (car list) 1) (addone (cdr list)))

which translates to

	:::scheme
	(cons (+ 3 1) (addone '()))

Now "addone" is called with the argument '(), because that is the *cdr* of '(3).

*4* - Aha! Now the base case *is* reached  because the argument is '() or in other words: null.

So the base case condition is executed:

	:::scheme
	(if (null? params) '())

which simply translates to:

	:::scheme
	'()

As you can see, this time we don't call the "addone" procedure again, but we return a value instead.
This means we now can walk back up the tree and start to give back a value to each procedure waiting.

The procedure waiting for its result was the previous one before our base case was met:

	:::scheme
	(cons (+ 3 1) (addone '()))

We can now fill in the result of the *addone*:

	:::scheme
	(cons (+ 3 1) '())

And the result of the sum:

	:::scheme
	(cons 4 '())

Now this procedure also has a value, being the cons cell *(4 . '())*, we can go one more up.
The one procedure waiting now is this one:

	:::scheme
	(cons (+ 2 1) (addone '(3)))

So this one now becomes:

	:::scheme
	(cons 3 (4 . '()))

We have yet another value, up to the next, final procedure:

	:::scheme
	(cons (+ 1 1) (addone '(2 3)))

Which now becomes:

	:::scheme
	(cons  2 '(3 4 '()))

This is the final, top most procedure, so this one simply can return its value, becoming:

	:::scheme
	'(2 3 4)

And there we have our result!

### Back to our egg

So let us put this recursion idea into practice with our *alist* builder. The procedure *build-alist* would look like this:

	:::scheme
	(define (build-alist params)
		(if (null? params) 
		'()
		(cons
			(cons
				(car params)
				(car (cdr params)))
			(build-alist (cdr (cdr params))))))

This looks much like the simple *addone* procedure, does it not? Okay, the last part is different of course, but the principle is the same.

First we put in our *base case*, which in many cases is to check for the empty list with the *null?* predicate. If the given *params* are not empty, the base case is not met and we perform a *cons*.
Let us take this cons line apart and see where the recursion happens:

	:::scheme
	(cons
		(cons
			(car params)
			(car (cdr params)))
		(build-alist (cdr (cdr params))))

We are consing two things together:

* A *cons cell* containing the *car* of the params (the first keyword) and the *car of the cdr* of the params (the value of the first keyword)
* A call to the *same procedure* but with the first keyword and the value of the first keyword stripped off (cdr (cdr params))

If we call this procedure with a list of keyword arguments:

	:::scheme
	(build-alist '(first: "string" second: "string" third: "string"))

CHICKEN will return us:

	:::scheme
	((first: . "string") (second: . "string") (third: . "string"))

Which is a list of cons cells instead of a list of keyword arguments, that is exactly what we want!

When the recursion ends, this procedure can return its value to the procedure that called it in the first place: our "build-params":

	:::scheme
	(define (build-params params)
		(if (keyword? (car params)) 
			(build-alist params)
			(list->vector (build-vector params))))	

Which does nothing more then give this result back to the procedure that called it, the *if* condition we where using to check if params existed while building our request object.

That is all there is to it for building an alist, but what if we have to build a vector using the "build-vector" procedure? Well, this turns out to be very much the same.

Let me present to you the code for "build-vector":

	:::scheme
	(define (build-vector params)
      (if (null? params)
	      '()
	      (cons
		    (symbol->string(car params))
			(build-vector (cdr params)))))

Pretty much the same, right? The only difference here is, again, the consing line at the end.
Since it is a vector we are building we do not need a keyword and its value, we simply have to build a list of items, one by one.

So the recursion here is the fact that we cons together the *car* of the params and call the procedure again with the *cdr*.
One odd thing you might notice is the *symbol->string* procedure that I have put in front of the *(car params)*.

This is needed to convert the symbols we input as arguments to the lambda to be converted into real strings.
If we would not do this, we would build a list containing symbol types which would make no sense at all to a JSON-RPC call.

Good. We now have our code almost completely finished, the full code for our "json-rpc-server" procedure and the procedures for building alists or vectors looks like this:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
    (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	  ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	  ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	  (else
	   (lambda (method . params)
	     (cond ((not (is-valid-method? method)) (server-setup-data-error "method" "can only be a string."))
		   ((not (are-valid-params? params)) (server-setup-data-error "params" "can only be a vector or an alist."))
		   (else
		    (remove null? (list (cons 'jsonrpc version)
							(cons 'method method)
							(if (null? params) '()
							    (cons 'params (build-params params)))
							    (cons 'id "1")))
				  input 
				  output))))))

	(define (build-params params)
		(if (keyword? (car params)) 
			(build-alist params)
			(list->vector (build-vector params))))

	(define (build-alist params)
		(if (null? params) 
			'()
			(cons
				(cons
					(car params)
					(car (cdr params)))
				(build-alist (cdr (cdr params))))))

	(define (build-vector params)
		(if (null? params)
			'()
			(cons
				(symbol->string(car params))
				(build-vector (cdr params)))))

There we have our whole mechanism for creating our request object. We have the procedure that sets up a connection and returns another procedure that builds the actually object using the small recursive procedures we just wrote.

The only problem now is that this code build the request object...and that is it. We still need to hand it over to Medea, the JSON<->CHICKEN converter we saw in chapter one.
Luckily for us (and thanks to [Moritz Heidkamp](http://wiki.call-cc.org/users/moritz-heidkamp "Moritz Heidkamp user page on the CHICKEN wiki") who made the Medea egg), this is yet another trivial task.

### Sending over the data

All we need to do is to call the sending procedure of Medea with our build request object and the output port we set when creating our connection with "json-rpc-server".
So first thing to do is to make sure we load in the Medea egg using the same *use* procedure as we did when loading in the SRFI-1 and Extras:

	:::scheme
	(use extras srfi-1 medea)

As we are now used to doing, let us create a small procedure that will send the data over to Medea:

	:::scheme
	(define (send-request request output)
		(write-json request output))

That is how simple it works. The "write-json" is a procedure from Medea that, well, takes the given object or "datum", converts it into JSON and sends it over to the output port given.

And now we need to call this procedure in our lambda, where the "request" argument will become the whole building process we have been seeing in this chapter:

	:::scheme
	(define (json-rpc-server input output #!key (version "2.0"))
    (cond ((not (input-port? input)) (server-setup-arguments-error "input port" "input-port" (get-type input)))
	  ((not (output-port? output)) (server-setup-arguments-error "output port" "ouput-port" (get-type output)))
	  ((not (is-valid-version? version)) (server-setup-arguments-error "version" "2.0" version))
	  (else
	   (lambda (method . params)
	     (cond ((not (is-valid-method? method)) (server-setup-data-error "method" "can only be a string."))
		   ((not (are-valid-params? params)) (server-setup-data-error "params" "can only be a vector or an alist."))
		   (else
		    (send-request (remove null? (list (cons 'jsonrpc version)
							(cons 'method method)
							(if (null? params) '()
							    (cons 'params (build-params params)))
							    (cons 'id "1")))
							output)))))))

Voila, we now call the procedure "send-request" where its first argument is the *whole* request object building process and the output port is the second argument.

### Defining and exporting

Good, we finished up the total code, *congratulations*!

Next up is to actually tell CHICKEN that this file is the source code of an egg.

First we load in the actual CHICKEN core at the beginning of our file:

	:::scheme
	(import chicken scheme)

Next we have to give this file a place inside our egg (eggs can consist of multiple source code files).

This is done trough the *module* procedure. This procedure takes several arguments, the first being the actual name of the module. In our case, the file we have been creating the past two chapters will contains the code for the *client* implementation. So I choose *json-rpc-client* as the name for this module file.

The next argument is the procedures you wish to export. When creating an egg with one or multiple files, you can choose to which of the procedures the user of your egg has access to. In our case, the users does need to use our predicate procedures or anything like that. There is only one procedure that they will need to use: our *json-rpc-server* procedure. So to scope their access, we can set this procedure to be exported.

The third argument is the *whole code* of our egg. Everything we have been writing up to this point goes into that procedure.

So, in the top of our egg file, we will end up with this:

	:::scheme
	(module json-rpc-client
		(json-rpc-server)
	    (import chicken scheme)
		
	    ... ; the whole egg code ; ...

    )

Mind the ending parenthesis at the end of your CHICKEN file to close off the last argument.

### Licensing

There is one more thing left before we can start dreaming of chapter three: setting the correct license.

It is important to set a correct license on your work, for protecting it, but also for making it possible for people to know if they can include it in *their* work.
The CHICKEN website has [a page](http://wiki.call-cc.org/eggs-licensing "Licensing page on the CHICKEN website.") containing all sorts of licenses you can use. Pick one carefully and put it in the top of your CHICKEN files.

### Looking ahead

While we probably have a functioning CHICKEN Scheme file, it is still not a real egg, it is merely one file containing a bunch of CHICKEN code and a module wrapper to give it a name and export the needed procedures.
As we have seen in chapter one, we now need to create some extra files so that CHICKEN will recognize this as a real module.

And one other *very important* thing we have to setup before you can release an egg is a *testing suite*.
It is not mandatory for releasing an egg, but it is highly recommended.
*Always* write tests for your code, this takes a little bit of your time, but can be a huge benefit in further development of your code.

Okay, I will leave you to rest now, take a deep breath and relax.

And as always...thanks for reading!
