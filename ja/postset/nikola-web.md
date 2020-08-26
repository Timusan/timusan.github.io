<!--
.. title: Nikola: the static blog engine A.K.A. How I build shisaa.be
.. slug: nikola-web
.. date: 2013/02/01 18:52:05
.. tags: web, python
-->

## Preface

First, a big thank you to Roberto Alsina, creator of Nikola, for giving this post [a warm welcome](http://nikola.ralsina.com.ar/blog/new-nikola-tutorial.html "Blog post about shisaa.be on Nikola's blog") and embedding it in the Nikola documentation!

When I designed shisaa.be I spend quit some time thinking about the "backend" to use. What system would generate the website for me...which CMS to use?
As I have quite some experience with Drupal, that would be a possible candidate...but being familiar with Drupal, is also knowing some hard-to-ignore facts:

* Drupal is written in PHP which is not a language I aspire much to program in.
* Drupal, besides being a powerful tool, is slow-as-hell, which is a typical burden of most all-purpose-cms-so-large-it-almost-becomes-framework systems out there.
* Drupal 8, the soon to be launched new and shiny version, is now build on top of another framework (symfony), so its a framework on top of a framework...which in most sane cases means it will become even slower.

So not Drupal then, what else is out there? Wordpress, Joomla, ...? Not likely to happen...ever...and they still are all hacked up in PHP.

Okay, no PHP driven systems...I seem to like Python more, so maybe Plone, Pylons, Pyramid, ...?

Maybe...

But there was still a nagging voice in the back of my head..."Do I really need a CMS"?

I mean, think about it, I'm the only author of this website, there are no external users who need to login and use fancy interfaces to put the word out there.
And...think of security...every single dynamic system on the internet is prone to all kinds of scary attacks...simply because their dynamic.
Oh, and performance, serving up a few plain old HTML files is light-years faster then dynamically throwing together a page per request (even with cache!).

So I decided to setup this whole site...wait for it...**statically**. But how?

When you decide to build your website statically you basically have two options:

1. Write each HTML document by hand
2. Use a static generator

If you choose option one, you have to consider the amount of work you are putting on your own shoulders. If your website or blog only consists of a few pages and gets updated two times a year, then good old handcrafted pages are the way to go. But if you tend to update the site frequently, writing each page from the ground up and maintaining all the menu and deep links can become a hell of a job...which would suck all the fun out of authoring your website.

That's where static generators jump in to help you. Once setup, these things will do the heavy lifting for you so you can concentrate on writing your blog, maintaining your company's website, putting pictures of your latest event online, etc...

"But I will need a lot of technical knowledge if I use these kind of systems", you ask. No, you will not.
While it is true that these systems always need a minimal of configuration (which system doesn't?), a good amount of eagerness can get you well underway!

## Step in Nikola

Besides the fact that I am a huge fan of Nikola Tesla, I quickly felt that [this](http://nikola.ralsina.com "Nikola homepage") Nikola would be my weapon of choice, its quite a fast and elegant system with a nice feature set...and its extensible. It also contains a balanced toolset at your service when building or authoring your site. It even is multilingual!

Go and read [Roberto Alsina's](http://nikola.ralsina.com.ar/handbook.html "Nikola handbook") handbook on Nikola to find out more about the reasoning behind this project and the beautiful features and tools available. On that site you will also find some pretty detailed guides on how to start building your site. Despite Roberto's best efforts of guiding you throughout Nikola's process, I sometimes ran aground, especially when theming, and had to go explore by myself.

So now I would like to take you on a short trip and show you how I used Nikola to setup, deploy and author this very website, mainly the basic setup and building your custom theme for the above stated reasons.

## The environment

Before we can dive in, we first have to setup Nikola itself...and to setup Nikola, we have to setup our Python environment.
Because many programs on Unix install various versions of python on your system, it is sometimes difficult to setup a clean workspace for you to run your projects in.
You really need a development sandbox where you can mess things up, install specific dependencies and use a specific versions of Python. For this sandboxing, I use the popular [Virtualenv](http://www.virtualenv.org/ "Virtualenv project page").

With Virtualenv you can do all of the above...you deploy a new virtual environment in a directory of your choice, point your shell to there and you can start building. First, install Virtualenv:

	:::bash
    $ pip install virtualenv
	
Now we can setup our virtual environment:

	:::bash
    $ virtualenv --python=python2.7 yourdir

Noticed that I told Virtualenv to use Python version 2.7, Nikola 5.3 *is* Python 3 compatible, but if you want to deliver a nice Google sitemap XML, you will have to stick with Python 2.x.
After Virtualenv is done setting up your sandbox, **go to that directory** and load this sandbox environment up into your current shell:

	:::bash
    $ source bin/activate

Voila, you will notice the current directory name in parenthesis in front of your shell, this means that all is well. Now...lets install Nikola!

	:::bash
    $ pip install nikola
	
This will give you Nikola 5.3 (on the time of writing) and install all the necessary dependencies for you. We can now initialize our site.

## The basics

To setup a new site, you can simply run:

	:::bash
	$ nikola init shisaa.be
	
Since Nikola 5.2 this creates a *really* empty site containing only the basic directory structure and an example configuration file.
You could go ahead and build this site, it works out of the box. Prior to 5.2 the init command would generate an empty site and fill it with demonstration data.
If you would like this behavior just set the "--demo" flag after the init command. 

When you enter your freshly created site directory you will notice the following structure:

	files
	galleries
	listings
	posts
	stories

Lets take a quick peek a what each directory means (and see some of Nikola's features along the way):

The *files* directory contains static assets that you want to have available regardless of the theme you use.
*Galleries* reveal the photo gallery feature of Nikola. Inside this directory you can create as many directories as you like, each being a separate gallery.
Next up is *listings*, this directory will contain source code listings. Here you can put your raw source code files and Nikola will create a listings page and syntax highlighted HTML files for each code file you put in there.
*Posts* get more to the heart of Nikola. This directory will keep all your post files in a chosen markup language.
The last directory is *stories*, it contains all the "static" pages (non-post pages) you wish to serve. Things like an *about* or *contact* page would fit nicely there.

But before we can understand how Nikola will treat all these directories we have to take a look at the configuration file...the controlling hub of your Nikola site.

## Create the basic configuration

Like we seen above, the basic site initialization gives you a fully packed *conf.py* file.
For the sake of learning...I would suggest you backup this conf.py and create an empty one.

Fire up your favorite editor and lets start building a fresh Nikola site configuration file.
Since Nikola is a Python application, the configuration file is also written in the Python language.
So the first thing we have to do is to load in some Python modules so Nikola can successfully parse the file:

    :::python
    from __future__ import unicode_literals
    import os
    import time

Next we can setup some basic information about our site, variables that we can use later in our templates:

	:::python
	BLOG_AUTHOR = "Tim van der Linden"
	BLOG_TITLE = "shisaa.be"
	BLOG_URL = "http://shisaa.be"
	BLOG_EMAIL = "tim@shisaa.be"
	BLOG_DESCRIPTION = "A blog about Programming, Japan and Photography"

Simple enough right? Now is a good time to save the file. It *has* to be named conf.py and *has* to be saved in the root of your Nikola site directory.

The next thing to do is to tell Nikola about the different content we would like to have processed. Because at heart Nikola is a blog engine, we will start there.

Static website generators follow roughly the same principle: You write your posts or pages in a markup language, define templates for these markup files and put in some static files like CSS or Javascript, then you let the generator do all the magic and output a directory with everything rendered in HTML. So first we define a place to put our blog posts markup files. In Nikola, this directory, by default, is called *posts*. You can actually call it anything you like, as long as your tell Nikola where to find it. 

So the next thing we have to enter in our configuration file is a hash table doing exactly that:
	
	::python
    post_pages = (
		("posts/*.md", "posts", "post.tmpl", True),
	)

This hash table is called *post_pages* and follows a certain syntax. First we define the directory where we will put our posts in and which files to look for (files with the *md* syntax in this case). Then we define the type of content Nikola will find there, in this case posts (you can also define pages here, but well see that later), The third entry is the template file to use when rendering. The last entry is a Boolean you can set telling Nikola wether to include these files in the sites RSS feed or not.

Next Nikola needs to know in which language we wrote down our static files. You can simply write down the *HTML* or use one of the many available markup languages that Nikola supports like *Markdown* or *reStructuredText*. For this we define another hash table called *post_compilers*:

	::python
	post_compilers = {
		"markdown": ('.md', '.mdown', '.markdown'),
    }

In my case I only use Markdown so that's the only one I define here. You can define more then one language here and based on the extension Nikola will use the correct compiler when translating it into HTML.

Quite fancy, right?

## Writing your post

Nikola has a command to start a new post:

	:::bash
	$ nikola new_post -f markdown
	
It will ask you some of the necessary meta data and the post text itself. Notice the *-f* flag, if you use multiple markup languages you will have to specifiy which language you want this new post to be written in.

A different way, and for me a more convenient one, is to create your post file manually and save it in your posts directory.

To start a post manually just open your favorite editor and save the file with a decent name under your posts directory.
The first thing we have to do in such a file is to declare some meta data Nikola will use to generate the HTML page, inner links and tags with.
Each line can contain one meta data attribute using a syntax that starts with two dots and a space. A default post or page would start with this:

	.. title: Your blog post title
	.. slug: any-unique-slug-you-like
	.. date: 2013/02/27 18:52:05
	.. tags: web, blog

After that you can start writing your post at your hearts content...in the markup language you defined in your *conf.py*.

## Building for the first time

Okay, a little bit early in the process, but we could already do a test build to see the effect of what we have done so far.

Just punch in the command and watch the process roll out:

	:::bash
	$ nikola build
	
Nikola will now create a directory called "output" where it will put your whole site structure.
A nice detail about the build process it also that it only builds the changes and everything related to that change.
Say you updated a blog post, then only that blog post will be re-rendered, including the files that have links pointing to that post....but nothing more then that.
This means that your initial build can take a few seconds, but after that is lighting fast!

If all went well, you can now take a peak at what was actually build.
To do this you could setup a web server and point it to the "output" directory...or you could use Nikola's build-in server.
From the same site root directory, start Nikola's build in server:

	:::bash
	$ nikola serve
	
Et voila...you can now point your browser to "http://localhost:8000" (the port is configurable in the config.py of your site) and watch the beauty unfold!
I usually open up a new terminal and have it run the Nikola server as long as I'm building so that in my main terminal I can punch in "nikola build" as much as I want to.
Remember that for each new terminal that you start you will have to set the source to your virtual environment again.

"But the website looks like Nikola's homepage!", you say. And that's correct.
What you see is the default theme that comes shipped with Nikola.
We want to make the site totally our own, so this also means a custom build theme.

So guess what...next thing up: *creating your theme!*

## Create a theme

As we have seen above, you can tell Nikola which template file (a file that has the *tmpl* extension) to use for each type of content you define in your *post_pages* hash table.
But where are these template files located? Well...In a theme.

Nikola will always search themes in the *themes* directory and load up the theme you defined in your *conf.py*:
	
	:::python
	THEME = 'shisaa'

Now Nikola will search for a directory called *shisaa* inside the *themes* directory.

"But wait!", you ask swiftly, "Before I did not declare a theme, but still there was the default one...what magic is happening here?"

Ah, glad you asked.

When you installed Nikola, you secretly also installed a few themes that come shipped with Nikola.

These themes reside in Nikola's sources. To get to them, you simply have to go to the *lib* directory where your Python is installed.
Remember, if you installed Python via a Virtualenv, then you have to open up that environment's *lib* directory.

In your *lib* you will find your Python version, open that one up and you will find a place called *site-packages*.
There you'll find the Nikola directory, and in there the *data* directory.
In that *data* directory sits a *themes* directory which holds all of the basic themes.
If you don't define any theme in your conf.py, it will use the one called *default*.

There, mystery solved!

Also nice to know: among the themes you'll find in there is a theme called *Orphan*. This is actually a stripped down theme that you can use to build upon.
So if you're tired of reading, just copy that to your themes directory, rename it and start tweaking.
Or just keep reading...and have some more fun with me!

You're still here? Good! Lets continue theming!

Theming in Nikola is done by the *Mako* or the *Jinja* theming engine, I chose *Mako* for the shisaa theme.
Mako needs a few directories inside your theme directory where you can store different type of files. You basically need three of them:

* assets
* templates
* messages

The *assets* directory will contain all the static files like CSS, Javascript, images, ... that are specific to your theme.
*Templates* is where you will put the actual *.tmpl* files.
And finally *messages* will contain the strings levering Nikola's multilingual capabilities. For now, we will only focus on English.

Lets start with the most exciting one...the *templates* directory.

Nikola uses a set of hardcoded template names it expects to find in that directory.
As explained in [the basic theming documentation](http://nikola.ralsina.com.ar/theming.html "Basic theming documentation") that comes with Nikola these are:

* base.tmpl
* index.tmpl
* post.tmpl
* listing.tmpl
* list.tmpl
* list_post.tmpl
* tags.tmpl

There are a few others (namely *gallery.tmpl*, *story.tmpl*) but that's for a little bit further up the road.

But before we start building each of these files, lets have a lightspeed glance at Mako:

Mako is a templating engine written in Python *for* Python. It uses a language that ties in very close with Python. A Mako template file can inherit from other Mako template files easily giving you the ability to separate certain parts of templating logic to keep everything tidy. You can inherit a file using the following syntax:
	
	:::mako
	<%inherit file="base.tmpl"/>

The next neat thing you will be using when inheriting are template blocks to include pieces of template code you want to use in your main template:

	:::mako
	<%block name="footer">
		<div id="footer">
			<p>This is a footer, just static html!</p>
		</div>
	</%block>

Now in your inheriting template file you can call this block like so:

	:::mako
	${footer()}

Besides inheriting you can use Python code almost seamlessly inside your template file.
To use Python code, start with *<%* and end with *%>* when your code block is finished:

	:::mako
	<%!
		text = "foobar"
	%>

And, of course, you also can use control structures to build loops or make simple decisions, caught exceptions, ...
I suggest you go and read the [Mako documentation](http://docs.makotemplates.org/en/latest/ "Mako documentation") to find out everything about this very powerful template language.
For now, lets get our hands dirty by creating a theme and start writing some Mako!

If you haven't already done so, create your themes directory in your Nikola site root and under that create a directory named after the theme your declared in *conf.py*.

The very first thing a theme needs are the translatable messages, even if your site is going to be in only one language, you still need to set the correct translation strings.
The message translations are stored in a directory called "messages" under you theme. So go ahead and create that directory.

Each language that you wish to support has its own translation file in the following syntax: **language-code***_messages.py*.
For the default English translation, create a file called *en_messages.py*.
This file *has* to be filled with a module import and a dictionary called *MESSAGES* containing a fixed number of items:

	:::python
	from __future__ import unicode_literals

	MESSAGES = {
		"LANGUAGE": "English",
		"Posts for year %s": "Posts for year %s",
		"Archive": "Archive",
		"Posts about %s": "Posts about %s",
		"Tags": "Tags",
		"Also available in": "Also available in",
		"More posts about": "More posts about",
		"Posted": "Posted",
		"Original site": "Original site",
		"Read in English": "Read in English",
		"Newer posts": "Newer posts",
		"Older posts": "Older posts",
		"Previous post": "Previous post",
		"Next post": "Next post",
		"old posts page %d": "old posts page %d",
		"Read more": "Read more",
		"Source": "Source",
	}

Even if you don't use all of these translations, you still need to declare them.

Next we can build our actual template files used for rendering out the final HTML.

**Warning: I altered each tmpl file more of less to fit my needs for shisaa.be. I do not make use of comments or multilingual features. Please review the tmpl files from the Orphan theme if you would like to see the original theme code!**

Because we need quite some HTML and a lot of Mako structures in our template files it can get very unreadable, very quickly.
To prevent this, it is a good practice to make use of Mako's *definition blocks*.
As we have seen a little bit earlier, Mako's gives you the ability to define blocks of "code" that you later can call by means of a simple function call.

"How do I go about doing that?", you ask me.

Well, lets just put our preaching to practice and create two files under your *templates* directory.
One is called *base.tmpl*, the other one is called *base_helper.tmpl*.
The *base.tmpl* is the main template file that will hold the skeleton of our HTML markup. This means the *head*, *html* and *body* tags.
The *base_helper.tmpl* is something that Nikola doesn't care for, but what you can use to store reusable definition blocks so you can keep your *base.tmpl* clean.
Actually, you can call this "helper" file anything you like (or create as many helper files as you like for that matter), but for clarity *base_helper.tmpl* is not such a bad choice.

Now lets uses these files together, shall we?

First, to be able to use the definition blocks you declare in your helper file inside *base.tmpl*, you first have to tell *base.tmpl* about it.
You can simply import the definition blocks by using Mako's "namespacing" attribute:

	:::mako
	<%namespace file="base_helper.tmpl import="*"/>

This line simply tells Mako to import all the definition blocks from the *base_helper.tmpl* file.
If you only want to import certain blocks from that helper file, simply remove the wildcard (\*), sum them up, separated by a comma, in the *import=""* attribute.
Put this line at the top of your *base.tmpl* file.

Before we start defining the blocks, lets create our skeleton in *base.tmpl*.
The shisaa skeleton looks somewhat like this:

	::html
	<!DOCTYPE HTML>
	<html>
		<html lang="${lang}">
		<head>
		</head>
		<body>
		</body>
	</html>

But this would result in beautiful yet totally white pages. We need to display actual content.

First, the *head*, it looks quite empty don't you think? It misses a title, some CSS, a favicon, ...
We could all just put it in the *base.tmpl* file, but we wanted it to be as uncluttered as possible.
So now may be a good time to edit our *base_helpers.tmpl* file and lets define our header block:

	:::mako
	<%def name="html_head()">
		<meta charset="utf-8">
		<meta name="title" content="${title} | ${blog_title}" >
		<meta name="description" content="${description}" >
		<meta name="author" content="${blog_author}">
		<title>${title} | ${blog_title}</title>
		<link type="image/x-icon" href="/assets/img/favicon.ico" rel="icon" />
		<link type="image/x-icon" href="/assets/img/favicon.ico" rel="shortcut icon" />
		<link href="/assets/css/reset.css" rel="stylesheet" type="text/css"/>
		<link href="/assets/css/theme.css" rel="stylesheet" type="text/css"/>
		<!-- Le HTML5 shim, for IE6-8 support of HTML5 elements -->
		<!--[if lt IE 9]>
		<script src="/assets/js/html5shiv.js" type="text/javascript"></script>
		<script src="/assets/js/html5shiv-print.js" type="text/javascript"></script>
		<![endif]-->
		%if rss_link:
			${rss_link}
		%endif
	</%def>

That looks a little bit more like what you would expect in a *head* tag, right?

As you can see we can use the variables we stored in the *conf.py* right here in the theme.
To use a variable you can just call it with the *${}* syntax. As you could also see with the *${lang}* variable for setting the correct locale HTML setting in *base.tmpl*.
Notice that some variables where not declare in your configuration (like *title*) but instead are fetched from the meta data inside your posts or pages markup files.

So now we have a neat header, but how to tell our *base.tmpl* to put this in the correct spot?
Simply by using the same syntax you used to render out variables, the only difference is that definition blocks should be called as if they where functions instead of simple variables.
This  means that instead of *${foo}* you have to call it with *${foo()}*.
So call this definition block in your *head* section:

	::html
	<!DOCTYPE HTML>
	<html>
		<html lang="${lang}">
		<head>
			${html_head()}
		</head>
		<body>
		</body>
	</html>

Now, what about the *body* section? Here needs to come the actual content from our posts, pages, etc...
We always want to display a side-wide title on screen, so lets add a nice and semantic *h1* tag containing the title and make it clickable to always go back to the homepage:

	:::html
    <h1 id="blog-title">
      <a href="${abs_link('/')}" title="${blog_title}">${blog_title}</a>
    </h1>

Next we need a placeholder so Nikola knows where to put all of the content we want to be rendered inside our skeleton.
Here we can introduce Mako's concept of *inheritance*.

The only thing we have to do is define a *block* where all of the other template files will refer to.
Under you page title, put in the following block:

	:::mako
	<%block name="content"></%block>

All of the other templates will refer to this block and they will render their output right at that spot.
These templates will start by inheriting from *base.tmpl*:

	:::mako
	<%inherit file="base.tmpl"/>

And they will include the same block with the same name, but with their specific templating markup:

	:::mako
	<%block name="content">
		...
	</%block>

Simple enough, right?

If its not totally clear yet, don't worry, we will come back to this later, first lets finish our *base.tmpl*.

The thing we typically want next after our content is a footer:

	:::mako
	${content_footer}

Now where does this *content_footer* variable come from? From your conf.py:

	:::python
	CONTENT_FOOTER = 'Contents &copy; {date} <a href="mailto:{email}">{author}</a> - Powered by <a href="http://nikola.ralsina.com.ar">Nikola</a>'

Put the above line in your configuration file and tweak as desired!

The only thing missing now is a menu.

On shisaa you will notice that I have a sidebar-style menu.
The links generated in the menu are also defined in the conf.py in a dictionary called...well...*SIDEBAR_LINKS*:

	:::python
	SIDEBAR_LINKS = {
		DEFAULT_LANG: (
			('/', 'Home'),
			('/archive.html', 'Blog),
			('/stories/about.html', 'About),
		),
	}

You'll notice that the menu is also multilingual, but since I only use one language, declaring the *DEFAULT_LANG* dictionary is enough.
In the dictionary you first declare the path and then the name you wish to have displayed in the menu.
The markup is build in our *base_helper.tmpl* file:

	:::mako
	<%def name="html_sidebar_links()">
		%for url, text in sidebar_links[lang]:
			% if rel_link(permalink, url) == "#":
				<li class="active"><a href="${url}">${text}</a>
			%else:
				<li><a href="${url}">${text}</a>
			%endif
		%endfor
	</%def>

"Whats all this?", you might wonder.

As we have seen in our small Mako introduction, you have the ability to use control structures and loops inside your templates.
Both are on stage in this little definition block.

First we loop over our current (or default) language in the *SIDEBAR_LINKS* dictionary.
For each item we find in the dictionary we check whether the link to be rendered is the current page, if so then set the "*active*" class. 
That's it, that's how our sidebar menu gets rendered.

When we put all of this together, the shisaa *base.tmpl* looks something like this:

	:::mako
	<%namespace file="base_helper.tmpl" import="*"/>
	<!DOCTYPE html>
	<html lang="${lang}">
		<head>
			${html_head()}
		</head>
		<body>
			<h1 id="blog-title">
				<a href="${abs_link('/')}" title="${blog_title}">${blog_title}</a>
			</h1>
			<ul id="sidebar">
				<li>
					${html_sidebar_links()}
				<li>
			</ul>
			<div id="content">
				<%block name="content"></%block>
			</div>
			<div id="footer">${content_footer}</div>
		</body>
	</html>

And our *base_helper.tmpl* looks like this:

	:::mako
	<%def name="html_head()">
		<meta charset="utf-8">
		<meta name="title" content="${title} | ${blog_title}" >
		<meta name="description" content="${description}" >
		<meta name="author" content="${blog_author}">
		<title>${title} | ${blog_title}</title>
		<link type="image/x-icon" href="/assets/img/favicon.ico" rel="icon" />
		<link type="image/x-icon" href="/assets/img/favicon.ico" rel="shortcut icon" />
		<link href="/assets/css/reset.css" rel="stylesheet" type="text/css"/>
		<link href="/assets/css/theme.css" rel="stylesheet" type="text/css"/>
		<!-- Le HTML5 shim, for IE6-8 support of HTML5 elements -->
		<!--[if lt IE 9]>
		<script src="http://html5shim.googlecode.com/svn/trunk/html5.js" type="text/javascript"></script>
		<![endif]-->
		%if rss_link:
			${rss_link}
		%endif
	</%def>

	<%def name="html_sidebar_links()">
		%for url, text in sidebar_links[lang]:
			% if rel_link(permalink, url) == "#":
				<li class="active"><a href="${url}">${text}</a>
			%else:
				<li><a href="${url}">${text}</a>
			%endif
		%endfor
	</%def>

Congratulations, we have setup the skeleton of our template!

Next we have to look into the different parts that will be rendered in our *content* block.
In the beginning of our templating adventure we saw that Nikola needs a minimal list of template files to successfully build your site.
We just covered *base.tmpl*, so whats still left?

* <s>base.tmpl<s>
* index.tmpl
* post.tmpl
* listing.tmp
* list.tmpl
* list_post.tmpl
* tags.tmpl

The *index.tmpl* file will render out content of the corresponding *index.html*.
On most blogs the index page contains a stream of the latest X posts that you wrote, or just contains the latest post with a pager to flip trough older posts.
I choose the latter, having a 20-scroll-page long index just doesn't look pleasing to me.

So this is my *index.tmpl*:

	:::mako
	<%inherit file="base.tmpl"/>
	<%namespace name="helper" file="index_helper.tmpl" import="*"/>
	<%block name="content">
		%for post in posts:
			%if (loop.first):
				<div class="index-post">
					<h2>${post.title(lang)}</h2>
					<p class="post-date">${post.date.strftime(date_format)} - </p>
					${helper.html_tags(post)}
					<div class="index-post-text">
						${post.text(lang)}
					</div>
					${helper.html_pager(post)}
				</div>
			%endif
		%endfor
	</%block>


At the top we setup the inheritance relation to our base.tmpl file. Next we load up the helper file. This helper will contain the logic for the tags and the pager rendering.
Finally we render out our basic markup. Because I only want the first post (most newest post) displayed, I use Mako's [loop context](http://docs.makotemplates.org/en/latest/runtime.html#loop-context "More information about Loop Context in Mako") to only show the first iteration of the loop.

The *index_helper.tmpl* looks like this:

	:::mako
	<%def name="html_tags(post)">
		% if post.tags:
			<ul class="tags">
				%for tag in post.tags:
					<li class="tag"><a href="${_link('tag', tag, lang)}">${tag}</a></li>          
				%endfor
			</ul>
		% endif
	</%def>

	<%def name="html_pager(post)">
		<ul class="pager">
			%if post.prev_post:
				<li class="previous">
					<a href="${post.prev_post.permalink(lang)}">&larr; ${messages[lang]["Previous post"]}</a>
				</li>
			%endif
			%if post.next_post:
				<li class="next">
					<a href="${post.next_post.permalink(lang)}">${messages[lang]["Next post"]} &rarr;</a>
				</li>
			%endif
		</ul>
	</%def>

Good, that's all for the index, one more off the list:

* <s>base.tmpl</s>
* <s>index.tmpl</s>
* post.tmpl
* listing.tmp
* list.tmpl
* list_post.tmpl
* tags.tmpl

Next logical step, our posts itself, the dedicated, individual post pages. These are handled by *post.tmpl*.
For every file we see on this list we can find the same pattern: they will all inherit from the *content* block declared in *base.tmpl*.
So lets pick up the pace a little bit. 

Below is my template for rendering a post:

	:::mako
	<%inherit file="base.tmpl"/>
	<%namespace name="helper" file="post_helper.tmpl"/>
	<%block name="content">
		<div class="postbox">
			${helper.html_title()}
			<small>
				${messages[lang]["Posted"]}: ${post.date.strftime(date_format)}
				${helper.html_tags(post)}
			</small>
			${post.text(lang)}
			${helper.html_pager(post)}
		</div>
	</%block>

And the helper file:

	:::mako
	<%def name="html_title()">
		<h1>${title}</h1>
		% if link:
			<p><a href='${link}'>${messages[lang]["Original site"]}</a></p>
		% endif
	</%def>

	<%def name="html_tags(post)">
		%if post.tags:
			${messages[lang]["More posts about"]}
			%for tag in post.tags:
				<a class="tag" href="${_link('tag', tag, lang)}"><span class="badge badge-info">${tag}</span></a>
			%endfor
		%endif
	</%def>

	<%def name="html_pager(post)">
		<ul class="pager">
			%if post.prev_post:
				<li class="previous">
					<a href="${post.prev_post.permalink(lang)}">&larr; ${messages[lang]["Previous post"]}</a>
				</li>
			%endif
			%if post.next_post:
				<li class="next">
					<a href="${post.next_post.permalink(lang)}">${messages[lang]["Next post"]} &rarr;</a>
				</li>
			%endif
		</ul>
	</%def>

By now, this all should look quite straightforward. We define three blocks in the helper file, where the last two are identical to what we used in the *index_helper.tmpl*:

* One to render the title
* One to render the tags and create a link to a page showing all other posts that contain this tag
* And a pager that can flip trough older and newer

Quite simple, right? 
Another one to cross off:

* <s>base.tmpl</s>
* <s>index.tmpl</s>
* <s>post.tmpl</s> 
* listing.tmp
* list.tmpl
* list_post.tmpl
* tags.tmpl

This next one is used to render the code listings feature we seen in the beginning of this post and looks like this:

	:::mako
	<%inherit file="base.tmpl"/>
	<%block name="content">
		<ul class="breadcrumb">
			% for link, crumb in crumbs:
				<li><a href="${link}">/ ${crumb}</a></li>
			% endfor
		</ul>
		${code}
	</%block>

We first render out a breadcrumb that resembles the path of the file in a hierarchical directory structure and after that we simply render the code. That's it!

* <s>base.tmpl</s>
* <s>index.tmpl</s>
* <s>post.tmpl</s>
* <s>listing.tmp</s>
* list.tmpl
* list_post.tmpl
* tags.tmpl

Finally we have three, almost identical templates: *list.tmpl*, *list_post.tmpl* and *tags.tmpl*. 
The main difference between them is the data that they have available.
*List.tmpl* has an *items* dictionary available containing general links, *list_post.tmpl* gets a *posts* dictionary containing all the posts and *tags.tmpl*, well, gets a dictionary with tag information.

The *list_post.tmpl* is used to render out the actual posts overview that belongs to a certain year (when browsing trough the archive). The template looks like this:

	:::mako
	<%inherit file="base.tmpl"/>
	<%block name="content">
        <!--Body content-->
        <div class="postbox">
			<h1>${title}</h1>
			<ul class="unstyled">
				% for post in posts:
					<li><a href="${post.permalink(lang)}">[${post.date.strftime(date_format)}] ${post.title(lang)}</a>
				% endfor
			</ul>
        </div>
        <!--End of body content-->
	</%block>

It is not needed to use a helper file because this template doesn't have so much functionality. It will just render out an unordered list of post titles.
You are, of course, free to pull anything from the *posts* you have available here and render it.

When you look at its counterpart, *list.tmpl*, you will find exactly the same code, but with *items* instead of *posts*:

	:::mako
	<%inherit file="base.tmpl"/>
	<%block name="content">
        <!--Body content-->
        <div class="postbox">
			<h1>${title}</h1>
			<ul class="unstyled">
				% for text, link in items:
					<li><a href="${link}">${text}</a>
				% endfor
			</ul>
        </div>
        <!--End of body content-->
	</%block>

This one if used to render the list of different years you have available.

Also *tags.tmpl* file uses *items* to contain all the information:

	:::mako
	<%inherit file="base.tmpl"/>
	<%block name="content">
		<div class="postbox">
		<!--Body content-->
			<h1>${title}</h1>
			<ul class="unstyled">
			% for text, link in items:
				<li><a class="tag" href="${link}"><span class="badge badge-info">${text}</span></a>
			% endfor
			</ul>
        <!--End of body content-->
    </div>
	</%block>

Phew, we have written out all the necessary template files.
This means Nikola now has enough information to actually render out each individual HTML page.

## Make a test run

So, are you ready for our second build? With your own theme this time? Punch it in!

	:::bash
	$ nikola build

If you still have a Nikola server running in another terminal, you can just refresh your page and see your new template in action.
Of course, we did not make any CSS, so everything will look quite like [CSS Naked Day](https://duckduckgo.com/?q=css+naked+day "Duck Duck Go search for CSS Naked Day") (which by itself is a good test).

In the beginning of this post we constructed the header and included some CSS files as I have used them on shisaa.be. Now go ahead and style your theme, correctly link up the files, and build again.

Now there you have it, a Nikola site up and running!

I hope I could make you feel the power you have at your fingertips when making your site with Nikola.
Its now ready to be tinkered with more and I encourage you to go and explore more features such as the gallery, multilingual, disqus integration, try a different markup language, ...

And as always...thanks for reading!
