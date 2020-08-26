<!--
.. title: Postgis, PostgreSQL's spatial partner - Part 1
.. slug: postgis-postgresqls-spatial-partner-part-1
.. date: 2014/06/12 10:00:00
.. tags: postgis, postgresql
-->

## Preface

In Dutch we have an expression that says "Van hier tot Tokio", which literally translated means "From here to Tokyo" and is used to indicate that something is *very* far or *very* difficult.
Unless you live in Japan, like me, then Tokyo is not *that* far actually....but you get the point. Tokyo is far, period.

But the question for today is...*how* far is it *exactly*? From where you are reading this right now...how far is Tokyo from you? How can you know?
You could of course just hop online and question your favorite search engine for help, or use something like Open Street Map to figure it out.

But that would be too simply, no? This would mean my post has to stop here, and, as some of you might know, it is difficult for me to write short blog posts. Sorry.

Also, you would miss out on all of the fun that is actually happening behind the screen when you question spatial search engines and that is against my belief: *know how the tools you depend on actually work*!

And, as the same some of you might know, I love PostgreSQL.

So, knowing that I cannot write short posts *and* I like PostgreSQL...what would you suspect would happen if you ask me how far Tokyo is from my current location?
You guessed it, simply use The Elephant to figure that out!

As I have showed you [before](http://shisaa.be/postset/postgresql-full-text-search-part-1.html "PostgreSQL full text search, chapter one."), PostgreSQL is capable of storing, matching and retrieving much more then boring VARCHAR or INT data types and it is designed to be extendable.
And extending is what the folks behind the *PostGIS* project did. To summarize, the PostGIS project extends PostgreSQL to store, match, manipulate and retrieve *spatial* data. It makes PostgreSQL a full-blown GIS.

The purpose of this series is to get your feet wet with PostGIS and to learn a thing or two about GIS itself.
In the first chapter, the one you are reading now, I would like to show you some fundamental GIS concepts: GIS Objects, standardization of GIS, geography and projections. 
We will not be doing any database action today I am afraid.

Then, starting from the second chapter, we will open up PostgreSQL, initiate a database to be PostGIS aware and start playing around.
We will look at a bunch of different database functions we have available and how the knowledge from this chapter maps to the actual database.
And we will of course be solving the question posed above: how far is Tokyo from your current location. 

Are you ready for a new PostgreSQL adventure?

*Note:* I will take you over all the following information in lighting speed. 
My intent is not to make you a GIS expert, but I do feel it is necessary to touch on a few important topics so you know why PostGIS is doing stuff the way it does.
This will hopefully make the actual database work from the next chapter more clear and spark some curiosity towards learning more about this topic.

## The data

Before we can do anything GIS related, we need to take a look at what kind of data we will be working with: the GIS objects.

### GIS Objects?

Geographic information system, or GIS in short, is merely the name of any system which can store, retrieve, generate, manipulate and visualize spatial data - the kind of data that represents objects in two or three dimensional space.

The GIS world is a world of standards, as with most computer sciences. These standards define what spatial data is and how we can work with it and is defined and maintained by the Open Geospatial Consortium or *OGC*.

Every system that wishes to work with GIS data, including PostGIS, should adhere to these standards.

### Simple Features

The OGC's standard for working with GIS data in SQL is defined in a OGC and *ISO* specification called *Simple Features*.

Simple Features defines how we can represent spatial objects, as you will see soon, but also defines how we can access and manipulate them.
You typically have available:

* Functions to *create* two dimensional spatial objects
* Functions to *alter* these objects
* Functions to *retrieve* and *describe* single or multiple objects
* Functions to *compare* and *measure* single or multiple objects

PostGIS has been certified by the OGC for its wide support of the Simple Features set.

### Well-known Text

The first and most important part that is defined in the Simple Features spec are the means by which we can represent spatial data.
I mean, we know how we can represent numbers or strings of text inside our database, but how do we represent something more abstract as a line, or a square?

Folks familiar with 2D drawing or 3D modeling software might already have a gut feeling of how to represent such data, and this gut feeling is right: you store coordinates.
If you wish to represent a line, you will only need to know the two end points of this line to be able to store, manipulate or visualize it.
The same goes for a square, though there you will need four coordinates.

And, as is always the case with standards, the OGC has devised two famous ways of representing these objects and their coordinates: Well-known Text or *WKT* and Well-know Binary or *WKB*.
These two are almost identical, only differing in the area of use.

WKT is a markup language which you can use to simply write down your objects and use it in queries. It is human readable.
However, if you wish to store it in a database or wish to perform matches on the data, it has to be stored in a defined binary format, the WKB format that is.

WKT can represent a wide range of objects from simple points to complex multi-polygons. The notation, however, stays roughly the same.
If you wish to represent a square, for example, you could use the *POLYGON* object:

	:::sql
	POLYGON ((4 1, 4 4, 1 4, 1 1, 4 1))

For people unfamiliar with the term "polygon", a polygon is a *closed*, *two dimensional* object with only *straight* lines. It has to have a minimum of three coordinates (points) thus giving it a minimum of three straight edges (making it, in that case, a triangle).

Let us take a deeper look at what is happening here. First, you will see we define a polygon object which you will need if you wish to represent closed, shape objects.
Next we define the four coordinates, the four corners of our square, laid out on a fictional grid of 4 by 4 units. There are two important notes to take about this coordinate listing:

First, the coordinates are all two dimensional and represent and X and a Y coordinate respectively.

Also, you do not see four but *five* coordinates. This is another rule from the spec that tells us that all polygon shapes *must* be closed.
To get a better visualization of this you could imagine a pen moving to each coordinate. To finish the loop you draw, the pen has to move back to the original coordinate.

The last thing to note is that the drawing direction of these coordinates is *counterclockwise*, as is with most computer defined drawing systems.
This means we put our pen on our grid at coordinate (4 1) and then draw *up* in a straight line to (4 4). Next we go *left* in a straight line to (1 4) and *down* in a straight line to (1 1).
Finally, we close the loop by drawing a straight line *right*, to the starting coordinate (4 1).

It is also perfectly possible to define more then one coordinate set when defining a polygon object.
A definition like this is perfectly legal:

	POLYGON ((8 1, 8 8, 1 8, 1 1, 8 1), (6 3, 6 6, 3 6, 3 3, 6 3))

This will create a square polygon with a size of 8 by 8, called the *exterior ring* and another square inside it with a size of 4 by 4.
Because this small square resides *inside* the area of the big square we call it the *interior ring* and, as a result, this small square will be interpreted by the standard as a hole in the bigger square.

To bring this even further, you can define as many holes in your exterior ring as you like, you simply have to make sure that the interior rings never touch each other and never go outside of the exterior ring.
The exterior ring is always derived from the first set of coordinates in your object definition.

The POLYGON object in the WKT standard also has a *MULTIPOLYGON* counterpart for when you wish to define a multiple, *non intersecting* set of polygon objects which, in turn, can have as many interior rings as you like.

Other objects we have available in the WKT standard are:

* POINT(0 0) to represent a point on a grid
* LINESTRING(0 0, 0 1) to represent a line. Note that a line can consist out of more then two coordinates.

All of these also have a *MULTIPOINT* and a *MULTILINESTRING* variant respectively.

As we have seen before, all of these objects are two dimensional, but PostGIS also partly supports a three and a four dimensional version of some of these objects.

*Note* that these extra dimensions are currently *not* in de specification and is a PostGIS specific extension on top of the features defined by the OGC.
Furthermore, if the OGC decides to standardize three of four dimensional objects, PostGIS will have to adapt its syntax to stay compliant.
We thus refer to this extended format not as WKT or WKB but as *Extended* WKT and *Extended* WKB or simply *EWKT* and *EWKB*.

To make our polygon object three dimensional, we could write it down like so:

	:::sql
	POLYGON ((4 1 2, 4 4 2, 1 4 2, 1 1 2, 4 1 2))

You can see that we now have three numbers per coordinate, the third one adds a *Z* or *depth* value.

A point gets even more fancier. If we wish to place a point in three dimensional space, we could write it down the same as we did with our polygon:

	:::SQL
	POINT (4 1 2)

The third parameter here being, again, a place on the *Z* axis.

But points can also have a *fourth* dimension which sounds fancy, but is nothing more then an extra reference we can ship with our coordinates.
This reference, also called a *linear reference*, is a number we can put in place that tell us where, along a linear path, the point we define resides.

It can be written down like this:

	:::SQL
	POINT (4 1 2 6)

Here we have four numbers, the last one being the linear reference or *M*.

With EWKT you also have the possibility to define a two dimensional object with a linear reference:

	:::SQL
	POINT M(4 1 6)
	
Here we have again three numbers, but to distinguish between the last number being *Z* or *M*, we have to reference *M* together with our point declaration.
There are more extensions defined in the EWKT and EWKB, but that is slightly off-topic, because, as I mentioned before, these are not standardized.
In most use cases you can simply use the standard WKT and WKB forms.

### What to use these objects for?

You now know what kind of objects we can represent using text and what we can, later along the road, insert into our PostGIS enabled PostgreSQL database.
But how do these points, lines and polygons help us measure distance or help us locate stuff?

First it is important to understand that all of the objects we have available will act as *proxies* to real world objects.
Take, for example, the point. A point can be used on a map to indicate a place, a spot so to speak, without defining shape or size.
When you wish to know where Tokyo is, a point will suffice on a global scale, you do not need nor want to know the exact shape of the metropolis.

However, if you would zoom in on our fictional map and you wish to see a part of the city the size of a few city blocks, you might be interested in the shapes of buildings, lakes, parks, etc.
These items that take up two *dimensional space* will be drawn with polygons that resemble the shape of the real world objects as close as possible.

Lines (or linestrings), finally, will almost always be used to represent roads, railroads, metro systems, etc. They many times represent actual *paths* one could travel along.

## Geometry and Geography

So you know that you can represent a place in the world with a simple point.
And as you also know, a point is defined like this:

	:::SQL
	POINT (10 20)
	
This would create a point that sits at coordinate (10 20). But, what does that mean?
How do these numbers relate to the *real world*? What *is* 10 or 20 anyway?

Well, first you will have to ask yourself the following question: Do I wish to be Cartesian or Geographical?

### Cartesian or Geographical

As you may or may not remember from your boring math lessons, a Cartesian system is a two dimensional flat grid with a X and a Y axis.
These axis go both positive and negative with the origin sitting exactly in the middle of the flat plane.

When working with GIS objects, we refer to this flat, Cartesian grid system as *Geometry*.
When, however we are working with measurements or objects related to the *real* earth we, in PostGIS, refer to these measurements as *Geography*.

Why? what is the difference? Well, to understand this, we have to take a step back, a step back into time that is.

When the first maps of the world where crafted, people truly believed the earth was flat (which it is not...for your information).
This meant that all charts that where drawn assumed we could simply place a grid comprised out of an X (length) and Y (height) axis across the drawing and from their measure distances between points. If you wish to know the distance between Paris and London, simply place two points on your map, take your *straight* ruler and measure the distance indicated.
Then factor in the chart's scale and you have your distance. You use *Geometry* or *Geometric measurements*.

However, after Copernicus nearly got his head chopped off telling people the earth was *not* flat, the chart drawing people gasped for air.
This meant their measuring technique was not correct. If the earth really was a sphere, then one could not simply wrap a grid around it and act as if everything was linear.
A sphere meant that there was a certain amount of distortion happening with their overlaying grid, and the measurements should encompass for those differences.

Even later in time, the chart drawing folk, who barely recovered from their first shock, where zapped again when people started to realize the earth was not a sphere either.
The globe turned out to be more of an egg shape, which, again, meant that measurement techniques had to be adjusted.

This was the birth of the *geographical* measurement system where cartographers devices a model called the *spheroid*.
A spheroid is a three dimensional object on which we can most accurately place points and measure real earth distances.
Each point on such a spheroid is define by a *latitude* and a *longitude*:

* Latitude is measured from the center of the earth (the hot place) in an angle up or down towards the surface
* Longitude is measured from the same hot center in an angle left or right towards the surface

Because both latitude and longitude represent an angle we express them as a *degrees* and we simply call the *geographical coordinates*.

Now, it is not quite convenient to have to carry around a three dimensional spheroid to find out where you are or to measure distance.
A classical old paper map is still more easy to bring along and more easy to work with.
But how do we go from a spheroid, which has the correct distortion, back to our old, flat, two dimensional geometrical map?

With *projection* or *map projection* to be more precise. We need to *project* the three dimensional spheroid system onto our two dimensional map.
This projecting is roughly done in three steps.

First we have to decide whether to take our spheroid as the base or a simpler sphere. A simpler sphere will yield less accurate results because it does not quite represent the correct curvature of the earth, but it does keep the maths behind the calculations simpler and thus can make for faster calculations. When choosing which shape we want, we also will have to define which *datum* we would like.

After choosing the base object and the datum that represents it, we have to transform the geographic system coordinates (latitude and longitude) to more standard X and Y coordinates to be used on a simple, flat, Cartesian plane. 

The last part is to find out to what ratio the final two dimensional surface is scaled compared to the original, base object (which represents the earth).

### Datum?

Before continuing, a word about datums.

As we said before, people agreed that the earth has a spheroid shape and that this model represents the earth most accurately.
We say "model" because the spheroid is something that is actually *defined* with math.

The math behind the spheroid model is what we call the *datum*. It is nothing more then a mathematical formula describing the shape.

Something we did not see is the fact that there actually are *many types* of spheroids out there. Each serving their own purpose and each with their own math aka datum.
Some spheroids are better to do measurements on a global scale, others are better for a more local "zoomed-in" level (continent, country, ...).

The reason we have to tell which datum (thus shape) our spheroid has, is because while latitude and longitude always represent degrees, they can have different meaning depending on the chosen datum.
If you use a datum that draws the spheroid a little bit "elongated" so to speak, then 1 degree longitude will cover slightly more distance then if the datum draws a more compact spheroid.

We will see more about datums in the next chapter, but it is an important part of GIS.

## Types of Projections

Something that might not be as obvious right now is the fact that going from our three-dee globe to a flat surface is a process of choices.
In an ideal world you wish to keep every aspect of your spheroid intact, meaning the proportions of the objects on the map are accurate everywhere, the shape of these objects is correct, the area covered by the objects is true and the distance between these objects is retained.
However, as it turns out, this is impossible on a two dimensional surface. You have to give up some of these properties to preserve others.

Throughout history there have been many attempts at creating projections that would keep as much of these aspects intact.

### Mercator projection

As a Belgian I should be most proud about this type of projection, since it was created by a fellow Flemish-man, around 450 years ago and it is a projection that is still being used today.
When a map is created with this type of projection we will get a comfortable and familiar view of the earth. 
A big advantage of this projection type is the fact that the shape of all objects are accurate.

The Mercator projection is most accurate around the equator, but the further you travel up or down, the more the map goes out of proportion.
Mercator used a cylindrical projection to unwrap the earth into a flat plane. Because of the nature of such a cylindrical projection, the areas more close to the poles become blown up to fit in a two dimensional world.

This distortion has caused quite some frowned foreheads in the last few decades and as a result people tend to abandon this projection, specially to project regions far from the equator.

### Mercator variants

To make up for the heavy distortions found in the original Mercator system, people have made two new Mercator projections.
The first that came about was called the *Transverse Mercator* which fixes the distortions around the poles, but introduces the problem that it will make for incorrect distance measuring.

To make up for this new problem, folks made yet another Mercator derivative: the *Universal Transverse Mercator*. This type of projection takes a whole new approach and uses its own coordinate system.
It introduces the concept of UTM zones. The earth is divided into roughly 60 zones and are each about 800 Km wide. The map that is rendered in each single zone uses the previous, Transverse Mercator projection to draw the actual map. A big advantage of this approach is the fact that we get a very constant distance measurement all across the globe.

Such a UTM coordinate looks quite different from our classic latitude/longitude or our X/Y version. I will give you a random UTM coordinate:

	:::SQL
	54 N 384524.5 3948304.4
	
The first number identifies one of the 60 UTM zones. The letter N show us in which hemisphere we should search this zone. These letters range from C to X (omitting I and O).
The first float tell us the *easting*, or X value, the last float tells us the *northing* or the Y value. Both these floats represent actual meters. 

Another important note to take about UTM is that it also acts as a framework for more localized UTM versions.
This means that each country or region could make its own maps, using smaller UTM zones to accurately represent their land, city, forest, etc.

### Lambert Azimuthal

This projection (also called the Lambert Equal-Area) is yet another approach as it uses a *disc* to map our spheroid to a flat surface.

The big advantage of this type of projection is the fact that it represent the area of objects very accurately and is true regarding distance calculation.
However, it fails when it comes to accurate shape representation for shapes get more and more distorted once you start moving away from the center of the disc.

The Lambert projection is one of the more accepted projections, right after the UTM.

### Plate Carrée

And then you have Plate Carrée.

This is one of the oldest projections out there and was invented around 1800 years ago.
In our little history story above, this projection came about when people thought the earth was rather flat.

It combines almost all disadvantages of previous projections:

* It does a terrible job in representing correct area
* It does not care about the shape of objects
* Distance measuring is way off

Despite the fact that this projection turns out to be so terrible, it is still quite commonly used today.
Well, not for navigation or distance calculation, obviously, but for illustrative purposes.

Many organizations across the globe use this simple projection to demonstrate statistical data, overlaid on this map.
Demographics, political info, zombie outbreak danger zones, ... .

As we also saw in our history lesson, the first charts used the Cartesian system quite literally and without much conversion, because the earth was flat anyway.
So in GIS systems, this means that this projection maps latitude and longitude *directly* to a X and Y coordinate without much conversion.

Because the conversion math is simple and calculations are few, this projection is among the fastest, but as you know now, at great cost.

### Other variants

There are numerous other variants out there that all have their advantages or disadvantages. To name a few more:

* The Robinson projection displays the earth not in a flat image, but in a cylindrical flat sphere. It show the world more accurately, but it fails when it comes to representing area and shape, especially near the poles.
* The Winkel Tripel projection is another popular projection type which has many parallels with the Robison one, but has less distortion.
* The Peirce quincuncial projection uses a technique to unwrap the earth spheroid into a square, much like you would peel an orange. These maps are not used much, for they are very heavy in calculations, but the technique is now widely used to present a spherical image, unwrapped into a square.
* The Goode homolosine projection is a projection developed as a teaching instrument in a frustrating answer to the heavily distorted Mercator projection. It is famous for its quite unique shape where the spheroid is unwrapped into a beast with four "legs".
* ...

## What do projections mean to us?

There are many more types of projections, but that would bore you to tears.

The fact that I keep going on about projections, geometry and geography is because later on, when working with PostGIS, you will need to make a decision about how you wish to combine all of these.

First it is very important to understand that geometry and geography are *two different data types* which PostGIS can store into PostgreSQL.

PostGIS is quite unique in the fact that it gives you the ability to work *directly* with our three dimensional spheroid (geography) and ignore the projections and their Cartesian Flat Land (geometry).
You will have the power to work with latitude and longitude and perform real world calculations, right out of the box.
This way of working, however, comes with a few trade-offs.

The first, and most obvious one: real, three dimensional spheroid geographical calculations will cost more computing time then the simpler, two dimensional geometry counterparts.
Another disadvantage of geography over geometry is the fact that PostGIS simply has *much* less native functions ready for you to use.

So depending on your use case, it might be a good idea to convert all your geographical data into geometrical ones.
This, however, requires knowledge about the projections we just saw for different projections will yield different results.

If you have two points with a latitude and longitude coordinate (thus being geographical data) and wish to know the distance between them using geometrical functions, you have to project these points on a flat surface thus converting them into a Cartesian system (the whole projection story we saw so far). 

As we will see in the next chapter, if you simply convert geography into geometry, PostGIS will project the geometry coordinates using the Plate Carrée, which may not be very desirable when you which to calculate distances as we will be doing later on. We have the ability to tell PostGIS to use a different projection when converting, but all come with merits and demerits.

You simply cannot do serious GIS work if you do not have at least a basic understanding of what is going on when projecting geography.
By reading through this chapter, I hope I have given you enough food-for-thought to go out and explore a bit more about these different projections.

## What is next?

Okay, I think we have covered enough for today. I do apologize for the rather theoretical nature of this first chapter, but believe me, you will need the knowledge.

Next time we will finally be looking at some actually PostGIS work and put some of this theory into practice:

* We will see how to GIS enable your PostgreSQL database
* We will look at how we can store geometry and geography
* We will actually put some points on the earth, draw some lines between them and perform some fun calculations
* We will take a look at how different projections will yield different results
* And finally, we will answer the question that started it all: How far is Tokyo from your current location?

And as always...thanks for reading!

<!--  LocalWords:  PostGIS GIS PostgreSQL
 -->
