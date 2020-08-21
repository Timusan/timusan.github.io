<!--
.. title: The Scheme programming language AKA The CHICKEN hens nest
.. slug: chicken-scheme
.. date: 2013/07/10 10:00:00
.. tags: chicken, scheme, json rpc
-->

## Introduction

I just recently finished up writing my first module (called an egg) for the Scheme dialect called CHCIKEN.
For me, this seemed as a good point in time to take you, my reader, on a journey to the world of Scheme.
Explain how I build this module, and try to show you what I have learned while writing it.

Ready? Then here we go...

## Preface (isn't that the same as an introduction?)

*Warning, cheese ahead...*

Imagine us walking down an street on  rainy evening.
You can hear thunder rolling far away in the clouds.
Next to us, there is something else walking, a stranger, an unknown.
All of a sudden I turn to you and say: "Ever heard of LISP or the Scheme programming language?".

You give me a blank stare.

"No? Well, then it's time that I introduce you to each other!"
I take a step back so you can see our stranger and shake its hand. 
"Reader, this is Scheme. Scheme, this is Reader. There, now you know each other!"

But you frown and say "It looks strange, almost...from another planet...and...It looks like my grandpa."

"Now, now, no need to judge so early! True that it may look a bit uncommon, and yes, it's older then the pilot episode of Lost in Space, but it is not limping, it is not begging to be put out of its misery, is it now?"

You look closer into it's luminescent little eyes and state in a monotone voice: "If I befriend it, people will frown upon me."

"Why?"

"Because people will think that I'm smug, a weeny, something better then they are..."

"Because you have befriended Scheme?"

"Yes! If I come too close to it, I may loose interest in my other friends, especially my good friend Pee-aich-pee."

"What did he ever do for you?"

"He gave me scripts...and...a shiny bandwagon to ride on...", your eyes turn watery. With a small voice you ask: "Does Scheme have a shiny bandwagon too?".

"No, I'm sorry, it does not. But it has something else..."

The Schemy little thing puts out its bony hands, holding out a small box.

"Take the box" I instruct you, "Take it and open."

Full with doubt you take the box and slowly open it. Your face becomes lit with a bright, angle-like light.
From the box rises and ancient figure. You start to wheep, tears streaming from your face.
"Its....its...the Lambda!", you burst out in tears of joy..."I see the light...I see it now...its so bueatifull!".

"Yes it is, yes it is." I pat your shoulder and the three of us walk of into the moonlight.

## Less cheesy preface (last preface, I promise)

Okay, that last part may have been a little bit over the top...

But before we dive into Scheme I want to take away the strange light that this language and its users are being put in by some people.
It is true that Schemers (people using Scheme) are smug-lisp-weenies (or smug-scheme-weenies), but that is not a *bad* thing.

From my personal experience, Schemers have been the best and most competent programmers that I have worked with so far.
And yes, they can be smug, really smug, but only when it is *justified* to be so. When another language fails horribly, Scheme most of the time has a much better alternative.
A good Schemer is smug by pointing out how it can be done better, safer and faster. A good Schemer will never, ever be smug on a personal level.

Scheme, or it's ancestor LISP, is a language that is designed around a totally different view then most, current programming languages.
It has been around long enough to see the birth of all the current hip and cool languages.
And it goes back all the way to the 1950's, quite close to the birth of the modern day computer. *And it is still here.*

Before I start sounding like and evangelist who should start a Church For Scheme, there are, of course, other nice and elegant languages.
To name a few: Python, Erlang, Pascal. Maybe even Perl or Ruby. There are dozens of decent languages out there.
But for some people, Scheme, LISP or Common LISP just have something *extra*. 

Now, at the time of writing, I have known about Scheme for a few years, but only just recently started to actually program in that language.
So I am in the privileged position of looking at the language from a noobish point of view, I can ask ignorant question, I can still take a long time before I will fully grasp Call With Current Continuation and I may call my fellow, venteran Schemers "smug-list-weenies- without getting smacked for it. Great!





