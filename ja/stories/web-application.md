<!--
.. title: Web Application
.. slug: web-application
.. date: 2014/10/06 20:00:00
-->

<a href="/ja/" class="back-to-home">< Back to home</a>

<a name="steps"></a>

One of shisaa.be's main pillars is developing custom web based software using only open source tools to bring your idea to life. No dreadful licensing fees, no nontransparent, black-box software.
All web applications will be programmed in Python: a very mature, object oriented programming language with a large community of professional developers.

When developing, shisaa.be does not reinvent the wheel and tries to apply existing and proven libraries and concepts to bring your project to the screen.
Using the Flask micro framework as the base ingredient, shisaa.be uses a bottoms-up approach to building your web application, avoiding bloat and creating only the parts you need.

Only the best libraries and tools are coupled together to deliver a fast, secure and slim web application:

* Twitter Bootstrap or Foundation as the graphical framework with a mobile first approach
* PostgreSQL as the robust, industry ready database to store all your information (SQLite if a lighter alternative is needed)
* PsycoPG2 as the database driver with direct, custom abstraction. No slow and buggy ORMs
* Jinja2 or LXML as secure templating languages
* LaTeX as the documentation language to write your developers manual
* Werkzeug as the routing library
* Nomad to provide upwards database migrations to keep your database synchronized with latest code changes
* PyTest and Nose to provide a proper unit test suite
* Mercurial (Hg) or GiT to keep all code under version control with atomic, descriptive commit messages

Furthermore, all the code is written with a test driven approach and thus all the application components will carry unit tests with a maximum coverage.
This does not only mean that every aspect of the code is tested on each change, but the code itself is clean, pythonic and atomically written.

To give you a better look at how shisaa.be helps you to create your web application, please take a look at the following steps from idea to finished project:
<br/>
<br/>

<h3 class="steps"><span>Step 1</span> Your idea, your business</h3>

<div class="steps-content">
  <p>Each web application is different and unique and requires careful planning. This planning starts by looking at what it is you want to actually build. What is the type of application, how complex will it be and what special requirements does it take. shisaa.be will help you to form your idea as best as possible, guiding you in the process.</p>
</div>

<h3 class="steps"><span>Step 2</span> Setup the application flow and wireframe models</h3>

<div class="steps-content">
     <p>After the idea and the requirements are crystal clear, the flow of the application has to be created.
        Creating the flow of an application means working out how the application should behave, which screens and messages should appear and which business logic has to be applied.</p>
     <p>Once the flow is decided, it will be put into a graphical flow chart which shows you what the application will do and give you a wireframe representation of each screen showing a rough draft of where text, buttons and input elements will go.</p>
     <p>This step is specifically intended to work out the full application before any code is actually written and to give you the opportunity to make changes in how the application works. Making changes in this stage of the development process is much more cost effective then doing so after the web application is already worked out in code.
</div>

<h3 class="steps"><span>Step 3</span> Designing the application</h3>

<div class="steps-content">
     <p>After the flow and wireframe models are finished and approved, it is time to actually design the basic theme and take a look at designing each screen and its elements in detail.</p>
     <p>Each major screen will be finished and presented to you for approval. A maximum of three alteration request can be made within the proposed design budget.</p>
     <p>We highly appreciate and encourage our customers input. If you have your own designer, or your own pictures, do not hesitate to present them during this step.
	Your designs, pictures and other elements will be carefully inspected and their usefullness for the web will be discussed with you.</p>
</div>

<h3 class="steps"><span>Step 4</span> Coding the web application</h3>

<div class="steps-content">
	<p>After the idea, flow and design is approved and ready, the actual coding can begin. Since each web application is different, each developing route will be even so.</p>
	<p>To help lift up the mystery and to keep you in the loop during the coding process, the web application will be build in phases. Each phase will be discussed beforehand, build and approved before starting on the next phase. This way you will know exactly what is already build, how it looks and feels and if the development is going in the right direction. This also allows you and shisaa.be to make last minute alterations before starting a new phase.</p>
	<p>shisaa.be has built countless applications, online and offline, and knows how important flexibility is, even during the coding process. Working in phases and keeping the communications channels open has been very welcomed by our current customers and has been a proven way to deliver solid, well coded applications, fully accommodating to the customers wishes.</p>
</div>

<h3 class="steps"><span>Step 5</span> Server setup</h3>

<div class="steps-content">
	<p>The final step, once the build is approved, is to setup your own, actual production server which will be used to serve your web application.</p>
	<p>To provide the most optimal production performance, each server will be delivered with a Debian OS and a server stack comprised out of NginX, as the reverse proxy server and uWSGI as the dedicated Python WSGI server that runs your WSGI processes in dedicated, emperor mode.</p>
</div>

<a name="cost"></a>
## What does it cost?

Since a web application is very custom and complexity and size varies greatly across applications, it is impossible to tell you how much your application will costs without knowing all the details. 
To give you a basic idea of what to expect, however, shisaa.be has prepared a starting price plan:

<table>
	<thead>
		<tr>
			<th>Step</th>
			<th>Time</th>
			<th>Price</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td class="item">Consulting</td>
			<td class="time">~ 2 weeks<sup>*</sup></td>
			<td class="price">35.000円<sup>**</sup></td>
		</tr>
		<tr>
			<td class="comment">Helping you to create your web application's idea, flow and wireframe screens. Good concept and structure will greatly benefit the final product.</td>
			<td></td>
			<td></td>
		</tr>
		<tr>
			<td class="item">Design</td>
			<td class="time">~ 2 weeks<sup>*</sup></td>
			<td class="price">40.000円<sup>**</sup></td>
		</tr>
		<tr>
			<td class="comment">Creation of the graphical layout, including the basic theme and each major screen with all the major elements.</td>
			<td></td>
			<td></td>
		</tr>
		<tr>
			<td class="item">Basic application development</td>
			<td class="time">~ 3 weeks<sup>*</sup></td>
			<td class="price">60.000円<sup>**</sup></td>
		</tr>
		<tr>
			<td class="comment">After the flow, wireframes and designs are ready and approved, the actual phased coding can begin.</td>
			<td></td>
			<td></td>
		</tr>
		<tr>
			<td class="item">Server deployment</td>
			<td class="time">~ 1 day<sup>*</sup></td>
			<td class="price">15.000円<sup>**</sup></td>
		</tr>
		<tr>
			<td class="comment">Once the coding is finished and the application approved, the production server will be setup and the web application deployed.</td>
			<td></td>
			<td></td>
		</tr>
		<tr class="total-row">
			<td class="info">
				* Times are rough estimates and vary depending on complexity, size and how fast your information reaches shisaa.be.<br/>
				** mentioned prices do not include VAT.
			</td>
			<td></td>
			<td class="total">150.000円<sup>**</sup></td>
		</tr>
	</tbody>
</table>

As mentioned before, the following table is based on small web applications and can be considered a starting price. Depending on your web application's size and complexity, price and times will vary.
Another important note is that each phase (consulting, design, development, server) will be billed separately, including the mentioned sub-phases in the development (coding) phase.

<a name="hosting"></a>
## Web application hosting

After your web application is up and running, shisaa.be offers a separate hosting and maintenance contract to run your application, keep the server and your application up-to-date and secure. Remember, shisaa.be does everything in house, even the full server management. This means the most optimal server installation for your web application. This contract will give you the following:

* Dedicated hosting of your web application
* You do not share your server with other customers, you get your own virtual private server
* Fully managed
* Full backup
* Security and feature updates of the core system (OS)
* Security and feature updates of the server stack (NginX, uWSGI, Python, ...)
* Monitoring and alerting of the most important processes
* Security updates of the web application itself

This contract carries the following, competitive pricing:

<table id="hosting">
	<thead>
		<tr>
			<th></th>
			<th>Pay Per Month</th>
			<th>Pay Per 6 Months</th>
			<th>Pay Per 12 Months</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td class="duration">6 Month duration</td>
			<td>15.000円<sup>*</sup></td>
			<td>75.000円<sup>*</sup> (1 month free)</td>
			<td>-</td>
		</tr>
		<tr>
			<td class="duration">12 Month duration</td>
			<td>15.000円<sup>*</sup></td>
			<td>75.000円<sup>*</sup> (1 month free)</td>
			<td>150.000円<sup>*</sup> (2 months free)</td>
		</tr>
		<tr class="total-row">
			<td colspan="4" class="info">
				* mentioned prices do not include VAT.
			</td>
		</tr>

	</tbody>
</table>

All the servers that shisaa.be uses for its customers are located in Tokyo so you enjoy the best speeds and performance for your web application.
Note that all servers come with a minimal, monthly royal data transfer limit of 2 Tb (2000 Gb). If this limit is crossed, extra costs will be billed.