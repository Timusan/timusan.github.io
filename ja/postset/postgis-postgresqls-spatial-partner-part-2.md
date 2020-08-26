<!--
.. title: Postgis, PostgreSQL's spatial partner - Part 2
.. slug: postgis-postgresqls-spatial-partner-part-2
.. date: 2014/06/18 10:00:00
.. tags: postgis, postgresql
-->

Welcome to the secoflynd part of our spatial story. If you have not done so, I advise you to go and read [part one](http://shisaa.be/postset/postgis-postgresqls-spatial-partner-part-1.html "Part one of this series.") first.

The first part of this series gives you some basic knowledge about the GIS world (GIS Objects, WKT, Projections, ...).
This knowledge will come in handy in this chapter.

Today we will finally take an actual peek at PostGIS and do some database work:

* We will see how we can create valid GIS objects and insert them into our database
* Next let PostGIS retrieve information about these inserted GIS objects
* Further down the line we will manipulate these object a bit more
* Then we will leap from geometry into geography
* Finally we will be doing some real world measurements

Let us get started right away!

## Creating the database

Before we can do anything else, we need to make sure that we have the PostGIS extension installed.
PostGIS is most of the time packaged as a PostgreSQL contribution package.
On a Debian system, it can be installed as follows:

	:::bash
	apt-get install postgresql-9.3-postgis-2.1
	
This will install PostGIS version 2.1 for the PostgreSQL 9.3 database.

Next, fire up your database console and let us first create a new user and database:

	:::SQL
	CREATE user gis WITH PASSWORD '10gis10';
	CREATE DATABASE gis WITH OWNER gis;
	
Not very original names, I know, but it states its purpose.
Next, connect to the *gis* database and enable the PostGIS extension:

	:::SQL
	\c gis
	CREATE EXTENSION postgis;
	
Now our database is PostGIS aware, and we are ready to get our hands dirty!

Notice that if you now describe your database:

	:::SQL
	\d

PostGIS has created a new table and a few new views. This is PostGIS's own bookkeeping and it will store which tables contain geometry or geography columns.

## Fun with Polygons

Let us begin this adventure with creating a polygon that has one interior ring, similar to the one we saw in the previous chapter.

Before we can create them, though, we have to create a table that will hold their geometrical data:

	:::SQL
	CREATE TABLE shapes (
		name VARCHAR
	);

Now we have a table named "shapes" with only a column to store its name. But where do we store the geometry?

Because of the new data types that PostGIS introduces (geometry and geography) and to keep its bookkeeping up to date, you can create this column with a PostGIS function named *AddGeometryColum()*:

	:::SQL
	SELECT AddGeometryColumn('shapes', 'shape', 0, 'POLYGON', 2);

Let us do a breakdown.

First, all the functions that PostGIS makes available to us are divided in groups that define their area of use. *AddGeometryColumn()* falls in the "Management Functions" group.

It is a function that will create a geometry column in a table of choice and adds a reference to this column to its bookkeeping. It accepts a number of arguments:

* The table name to where you wish to add the column
* The actual column name you wish to have
* The SRID
* The WKT object you wish to represent
* The coordinate type you desire (2 means XY)

In the above case we thus wish to add a geometry column to the "shapes" table. The column will be named "shape". The geometry inserted there will get an SRID of 0 and will be of object type POLYGON and have a normal, two dimensional coordinate layout.

### SRID?

One thing that you might not yet know from the above function definition is the *SRID* or *Spatial Reference ID* and is a *very* important number when working with spatial data.
Remember in the last chapter I kept on yapping about different projections we had and that each projection would yield different results?
Well, this is where all this information comes together: the SRID.

Our famous OGC has create a lookup table containing a whopping *3911* entries, each entry with a unique ID, the SRID.
This table is called *spatial_ref_sys* and is, by default, installed into your PostgreSQL database when you enable PostGIS.

But hold on, there is something I neglected to tell you in the previous chapter: the European Petroleum Survey Group or EPSG.
The following is something that confuses many people and makes them mix-and-match SRID and EPSG ID's. I will try my best not to add up to that confusion.

### EPSG

The EPSG, now called the OGP, is a group of organizations that, among other things, concern themselves over cartography.
They are the world's number one authority that *defines* how spatial coordinates (projected or real world) should be calculated.
All the definitions they make get and accompanying ID called the EPSG ID.

The OGC maintains a list to be used inside databases (GIS systems). They give all their entries a unique SRID.
These entries refer to *defined* and *official* projections, primarily maintained by the *EPSG* which have their own EPSG ID and unique name.
Other projections (not maintained by the EPSG) are also accepted into the OGC SRID list as are your own projections (if you would feel the need).

Let us poke the spatial reference table and see if we can get a more clear picture.

If we would query our table (sorry for the wildcard) and ask for a famous SRID (more on this one later):

	:::SQL
	SELECT * FROM spatial_ref_sys WHERE srid = 4326;

We would get back one row containing:

* srid, which is the famous id
* auth_name, the name of authority organization, in most cases EPSG
* auth_srid, the EPSG ID the authority organization introduced
* srtext, tells us how the spatial reference is built using WKT
* proj4text, commands that drive the proj4 library which is used to make the actual projections

And as you can see, both the "srid" column and the "auth_srid" are identical. This will be the case with many entries.

I should also tell you that this huge list of SRID entries mostly consists of dead or localized projections.
Many of the projections listed are not used anymore, but where popular some time in history (they are marked deprecated), or are very localized. 
In the previous chapter I mentioned that the general UTM system, for example, could be used as a framework for more localized UTM projections.
There are hundreds of these local projections that only make sense when used in the area they are intended for.

### Simple Features Functions

As I have told you before, the functions that PostGIS makes available are divided into several, defined groups. The functions themselves are too defined, not by PostGIS but by the Simple Features standard maintained by the *OGC* (as we saw in the previous chapter).

There are a total of 8 major categories available:

* Management functions: functions which can manipulate the internal bookkeeping of PostGIS
* Geometry constructors: functions that can create or construct geometry and geography objects
* Geometry accessors: functions that let us access and ask questions about the GIS objects
* Geometry editors: functions that let us manipulate GIS objects
* Geometry outputs: functions that give us various means by which to transform and "export" GIS objects
* Operators: various SQL operators to query our geography and geometry
* Spatial relationships and measurements: functions that let us do calculations between different GIS objects
* Geometry processing: functions to perform basic operations on GIS objects

I have left a few categories out for they are either not part of the Simple Features standard (such as three dimensional manipulations) or beyond the scope.
To see a list of all of the functions and their categories, I advise you to visit the PostGIS reference, [section 8](http://postgis.net/docs/reference.html).

Let us now do some fun manipulations and use some of the functions from these categories, just to get a bit more familiar with how it all works together.

If you inserted the last SQL command which makes the geometry column, you should have gotten back the following result:

	:::SQL
	public.shapes.shape SRID:0 TYPE:POLYGON DIMS:2

This tells us we created the "shape" column in the "shapes" table and set the SRID to 0.
SRID 0 is a convention used to tell a GIS system that you currently do not care about the SRID and simply want to store geometry with an arbitrary X and Y value.

Let us now insert the shape of our square. To insert a polygon into your column, you could use various functions. One of these functions is *ST_GeomFromText()*:

	:::SQL
	INSERT INTO shapes (name, shape) VALUES (
		'Square with hole',
		ST_GeomFromText('POLYGON ((8 1, 8 8, 1 8, 1 1, 8 1), (6 3, 6 6, 3 6, 3 3, 6 3))', 0)
	);
	
This now inserts our polygon and gives it a name. The *ST_GeomFromText()* function enables us to enter our polygon object using WKT.
This function also accepts a second, optional parameter which is the SRID by which we wish to work.
The category of this function is called *Geometry Constructors*.

You know this polygon has two rings, the exterior and the interior. Let us now ask PostGIS to return only the line that represents the exterior ring:

	:::SQL
	SELECT ST_ExteriorRing(shape)
		FROM shapes
		WHERE name = 'Square with hole';

And we get back:

	:::SQL
	0102000020E6100000050000000000000000002040000000000000F03F00000000000020400000000000002040000000000000F03F0000000000002040000000000000F03F000000000000F03F0000000000002040000000000000F03F
	
Oh my...that is not what we expected. But yet it is correct. This is how PostgreSQL stores geometry/geography.
The result is correct, yet unreadable to us humans. 

If we wish to get back a readable WKT string, we have to convert it using one of the conversion functions:

	:::SQL
	SELECT ST_AsText(ST_ExteriorRing(shape))
		FROM shapes
		WHERE name = 'Square with hole';

And we get:

	:::SQL
	LINESTRING(8 1,8 8,1 8,1 1,8 1)

Aha, that is more like it! This we can read!

We used the *ST_ExteriorRing()* which falls under the *Geometry Accessors* category and the *ST_AsText()* function which resides in the category *Geometry Outputs*.

Okay, now we wish to know the interior ring:

	:::SQL
	SELECT ST_AsText(ST_InteriorRingN(shape,1))
		FROM shapes
		WHERE name = 'Square with hole';

The result:

	:::SQL
	LINESTRING(6 3,6 6,3 6,3 3,6 3)

Notice that the function *ST_InteriorRingN()* requires you to give the integer of which ring you wish to get, starting from 1.
As we have seen before, polygon objects can have multiple interior rings, but only a single exterior one.

Next let us ask all the information about what makes up the shape:

	:::SQL
	SELECT ST_Summary(shape)
		FROM shapes
		WHERE name = 'Square with hole';

And get back:

	:::SQL
	Polygon[B] with 2 rings+
      ring 0 has 5 points  +
      ring 1 has 5 points

Ohh, that is pretty cool. We get back a human readable string that explains to us how this particular piece of geometry is build.

Let us now add another polygon:

	:::SQL
	INSERT INTO shapes (name, shape) VALUES (
		'The intersecting one',
		ST_GeomFromText('POLYGON ((14 1, 15 8, 7 8, 7 1, 14 1))', 0)
	);
				
This is a polygon that will *intersect* with part of our previous polygon.
Let us ask PostGIS if these polygons really intersect each other:
	
	:::SQL
	SELECT ST_Intersects(a.shape, b.shape)
		FROM shapes a, shapes b 
		WHERE a.name = 'Square with hole' AND b.name = 'The intersecting one';
	
If all is well, this will simply return *TRUE* if they intersect and *FALSE* if they do not. In this case, it will return *TRUE*.

The counterpart of our intersect function is *ST_Disjoint()*:

	:::SQL
	SELECT ST_Disjoint(a.shape, b.shape)
		FROM shapes a, shapes b 
		WHERE a.name = 'Square with hole' AND b.name = 'The intersecting one';	

Which will return *FALSE* in our case.

Let us now add a third polygon which does not intersect our previous two:

	:::SQL
	INSERT INTO shapes (name, shape) VALUES (
		'The solitary one',
		ST_GeomFromText('POLYGON ((20 20, 20 40, 1 40, 1 20 ,20 20))', 0)
	);	
	
This polygon will reside well "above" the other two and does not share any space.
Let us now see how far this polygon resides from our first polygon, the one with the hole:

	:::SQL
	SELECT ST_Distance(a.shape, b.shape)
		FROM shapes a, shapes b 
		WHERE a.name = 'Square with hole' AND b.name = 'The solitary one';	

This returns us the number "12", which means they are 12 units apart.
And remembering the definition of both shapes, the first shape is 8 units tall and the second shape starts at unit 20.
This indeed leaves a gap of 12.

Nice! We have just measured the distance between two objects in a spatial database!

Hmmm, this may mean we are getting closer to knowing the distance to Tokyo...but not yet, we need to play a bit more first.

PostGIS also has the ability to manipulate geometry. Let us, for example, try to move our solitary polygon even further away using the *ST_Translate()* function under the *Geometry Editors* category:

	:::SQL
	SELECT ST_Translate(shape, 0, 10)
		FROM shapes
		WHERE name = 'The solitary one';

The *ST_Translate()* function will accept the to-be-altered geometry and accepts an X, Y and an optional third dimension.

Running this query will give us a binary representation of a *new* piece of geometry. The original geometry is not altered.
So how can we actually move the geometry that resided in the database?

Simply by using SQL:

	:::SQL
	UPDATE shapes
		SET shape = ST_Translate(shape, 0, 10)
		WHERE name = 'The solitary one';

Let us now check the new distance:

	:::SQL
	SELECT ST_Distance(a.shape, b.shape)
		FROM shapes a, shapes b 
		WHERE a.name = 'Square with hole' AND b.name = 'The solitary one';
	
And get back:

	:::SQL
	22
	
Aha! Nice! It has now moved ten units upwards.

Now let us alter the distance once again, but this time we will scale the polygon down:

	:::SQL
	UPDATE shapes
		SET shape = ST_Scale(shape, 0.5, 0.5)
		WHERE name = 'The solitary one';	

If we now check the distance, we will get back:

	:::SQL
	7
	
Wow, we have gone from 22 to 7, how did that happen?

Well, it is important to know that the *ST_Scale()* function currently only supports scaling by multiplying each coordinate. This means that the polygon will not only become smaller or bigger, but will also translate as a result. To know exactly how our new, scaled version of our polygon looks, we can use the *ST_Boundary()* function which shows us the outer most linestring:

	:::SQL
	SELECT ST_AsText(ST_Boundary(shape))
		FROM shapes
		WHERE name = 'The solitary one';

And we will get:

	:::SQL
	LINESTRING(10 15,10 25,0.5 25,0.5 15,10 15)
	 
If we compare that to the same result before scaling (which I handily made ready for you):

	:::SQL
	LINESTRING(20 30,20 50,1 50,1 30,20 30)	 

You can see that each value in each coordinate simply was divided by 2.
This also clarifies why our polygons are now only 7 units apart (first square stops at 8, the scaled square start at 15).

Okay, okay, I guess we have played enough now.
We have seen a small glimpse of the operations you can do on GIS data within PostGIS and seen that PostGIS makes all of this work fairly easy.

I guess we can now take it one step further and start to actually look at some geography!

## Fun with the earth

Up until now we have been working with an SRID of *0*, which means *undefined*, inside a *geometry* column, meaning the data was of type "geometry".
Now we want to go out and explore the actual earth, which means we wish to continue in a *geographical coordinate system*.

This brings us at a crossroad of choices. First, you will need to ask yourself the same question we pondered in chapter one: you wish to work with geometry or geography?

On the one hand we know that geographical measurements are expensive calculations, but most accurate for they are unprojected.
On the other hand GIS convention tells us that in any case, we should continue in a geometrical or Cartesian system, simply because...well...it is a convention.

So what do we do?

It all depends on your specific use case.

When working on a "small" scale, say, part of North America, it would make sense to not use geography.
Instead, you could (and should) work in a geometrical system using a very accurate projection with SRID 4267 (datum *NAD27*) or SRID 4269 (datum *NAD83*) which are both local UTM variants for North America. 

Depending on which region you work in, chances are high you have several local projections with their own datum and coordinate system, ready to use.
They are very accurate and less expensive to use then direct geography.

For us, however, we will be working on a large scale, for we want to measure a distance that covers much of the globe. You cannot use a local projection or local datum for that.

In such a case you, again, are presented with two options.
You could either neglect the convention and simply use geographical data and functions or be nice and adhere to what is agreed upon and work in a Cartesian system.

We will be doing both and we will use the common SRID *4326*.
This *very* popular SRID is by heart geographical, for it uses the geographical coordinate system, but can also be used with geometrical data. Confused?

Join the club.

Let me try to clarify.

First, the authority of this SRID is the EPSG and the EPSG ID is identical to the SRID.
It uses a popular *datum* (remember chapter one) called *WGS 84* and is referred to as *unprojected* for it is a geographical representation.
This datum is one that is used in GPS systems and is often referred to as a *word wide datum*.

When you store objects with an SRID of 4326, you are storing them using geographical coordinates aka latitude and longitude.
This in contrast to, for example, the former SRID's, like 4267 or 4269, which store their coordinates in UTM values.
When you do measurements between two objects carrying this SRID you have two options. You can either do a geographical or a geometrical measurement.

With a geographical measurement there will be no projection and the system will use the WSG 84 datum (the spheroid) to calculate the distance, in three dimensional space.
As we have seen before, such a calculation is more expensive and unconventional.

With a geometrical measurement, your geographical coordinates have to be *projected* on to a flat Cartesian or *geometrical* plane.
This is done automatically when you ask PostGIS to measure distance using one of the more common geometrical functions.
When projecting, all GIS systems will use the *Plate Carrée* projection which means they will use the stored latitude and longitude coordinates directly as an X and Y value.

Let us see this story in action. First we can take a look at the more native geography data. Let us clean our shape table first:

	:::SQL
	SELECT DropGeometryColumn('shapes', 'shape');

Here we use the *DropGeometryColumn()* to remove this column from out "shapes" table. Now clear the table:

	:::SQL
	TRUNCATE shapes;

Next add a new geography column:

	:::SQL
	ALTER TABLE shapes ADD COLUMN location geography('POINT');

We create a new *geography* column with the name *location* in our "shapes" table. We will only be storing Point types.

Notice that the syntax is different and that here we use plain SQL as opposed to the *AddGeometryColumn()* function from before.
Since PostGIS 2 it is possible to create and drop both geometry and geography columns with standard SQL syntax.

If you wish to rewrite our "shape" column addition from the beginning of this chapter, you could write it like this:

	:::SQL
	ALTER TABLE shapes ADD COLUMN shape geometry('POLYGON',0);

Looks more native and simple, no? Sorry to tell you this so late in the adventure, but now you know the existence of both the functions and the more native SQL syntax. Both will also keep the PostGIS bookkeeping in sync.

Also, for fun, you could do a describe on the table:

	:::SQL
	\d shapes
	
And get back:

	:::bash
	 Column  |         Type          | Modifiers 
    ----------+-----------------------+-----------
    name     | character varying     | 
    location | geography(Point,4326) | 

As you can see, the column is of type *geography* and automatically gets the famous SRID 4326.

Good, let us now try and find an answer to our famous question, How far is Tokyo from my current location. You will be surprised how trivial this will be.

First, as you might suspect, since we are only interested in a point on the earth and not the shape of your location nor Tokyo, we will suffice with a Point object.
Next we will need to insert two points into our database, your location and the center of Tokyo, both in geographical coordinates.

### Finding Your Location

This means you need to find out your exact latitude and longitude of the place you are at right now.

This could, of course, be done in a myriad of ways: using your cell phone's GPS capabilities, using your dedicated GPS device or using an online map system.
I will choose the latter and will be using OpenStreetMap (what else?) to locate my current position.

Open up your favorite web browser and surf to [openstreetmap.org](http://openstreetmap.org).
Once there, punch in your address or use the "Where Am I" function. This would give you a point on the map and in the search bar on the left your latitude and longitude coordinate.
Take this coordinate and save is as point data into your fresh column:

	:::SQL
	INSERT INTO shapes VALUES ('My location', ST_GeographyFromText('POINT(127.6791949 26.2124702)'));
	
The point I am inserting reflects central Naha, the main city of the Okinawa prefecture. Not my current location, but it serves as an illustrative point.

*Note* that PostGIS expects a longitude as X and latitude as Y. This is many times reversed as what you get back from other sources.

Now you can insert the location of Tokyo, which I conveniently looked up for you:

	:::SQL
	INSERT INTO shapes VALUES ('Tokyo', ST_GeographyFromText('POINT(139.7530053 35.6823815)'));

Ah, nice! Okay, are you ready to finally, after all the rambling we went through, know the distance?
You already know the syntax, punch in the magic:

	:::SQL
	SELECT ST_Distance(a.location, b.location)
		FROM shapes a, shapes b 
		WHERE a.name = 'My location' AND b.name = 'Tokyo';

In the case you would live in the exact cartographic center of Naha, you will get back:

	:::SQL
	   st_distance    
    ------------------
     1557506.28103692

Yeah! This, my lovely folk, is how far you are from Tokyo, at this very moment.

But what is this number you get back? 

The result you see here is the distance returned in *Meters*, meaning, from the point I inserted as "My location", I am 1557506.28 Meters or *1557.50628 Kilometers* from Tokyo.

Very neat stuff, would you not say? PostgreSQL just told us how far we are from Tokyo, *awesome*!

But wait, we are not finished yet. We have now done the most accurate, real geographical distance measurement using expensive geographical calculations.

There is an "in-between" solution before we jump to geometry. PostGIS gives us the ability to replace our spheroid datum with the more classical sphere.
The latter has much simpler calculations, but can still return more accurate results them some of the projections.

To redo our calculation from above with a sphere, simply set the spheroid Boolean, a third and optional parameter to the *ST_Distance()* function, to *False*:

	:::SQL
	SELECT ST_Distance(a.location, b.location, False)
		FROM shapes a, shapes b 
		WHERE a.name = 'My location' AND b.name = 'Tokyo';

The result:

	:::SQL
       st_distance    
    ------------------
    1557886.68227339

Which is a total distance of *1557.886 Kilometers*, a difference of around 300 Meters.

Let us now repeat this story, but use *geometry* instead. Let us do it the GIS conventional way.

We do not need to recreate our column as a geometry column and insert our data again. We could cheat a little.
PostGIS together with PostgreSQL has the unique capability of *casting* data from one type to another.
So without recreating anything, we could simply cast our geography data into geometry *on the fly* and see what happens.

*Note* that casting, while very convenient for quick checks, can render an index totally mute.
It is therefor important to think ahead and decide if you want to work with geometry or geography, then create the correct column type and use this *without* casting.

But for our quick and dirty queries, this is fine. Let us continue:

	:::SQL
	SELECT ST_Distance(a.location::geometry, b.location::geometry)
		FROM shapes a, shapes b 
		WHERE a.name = 'My location' AND b.name = 'Tokyo';

In this query, we cast (*::*) the geography data inside the "location" columns into geometry.
Now we get back:

	:::SQL
	   st_distance    
    ------------------
	15.3445794209231

Hmm, that is a different result all together. It looks like a much smaller number then before. What is happening?

We just casted our geography to geometry, this means PostGIS will now use a Cartesian system or *projection* to calculate the distance in a linear way.
When using the distance measuring function *ST_Distance()* on geometry, it will return not meters but the distance expressed in the units the original data was stored in.
Since our data is stored with SRID 4326, its units are latitude and longitude. The value you get back is thus *degrees*.

In the case of the Naha location, this will be *15.344* degrees from Tokyo.

For our human brain this is difficult to imagine, a result in Meters is much more easy to comprehend. So, let us transform this degree value into a metric value.
	
It is an estimation that one planar degree (in our Cartesian system) equals 111 KM. So the distance now becomes 15.344 degrees times 111: *1703 Kilometers*.
That is a difference of about 145 Kilometers. 

The reason this difference exist is of the projection we are now using. As we have mentioned a few times before, when going from data containing SRID 4326, PostGIS will automatically use the infamous *Plate Carrée* projection. This projection, as we have seen before, is the *least* accurate for something like distance measuring.

So let us poke this projection mechanism and try a different, more accurate one, the Lambert, which carries SRID *3587*.

To change the projection PostGIS will use, we can use the *ST_Transform()* function which casts objects to different SRIDs.
Note that *ST_Transform()* only works for geometry objects, so we have to continue to cast our geography location to be able to use them in this function.

	:::SQL
	SELECT ST_Distance(ST_Transform(a.location::geometry, 3587), ST_Transform(b.location::geometry, 3587))
		FROM shapes a, shapes b 
		WHERE a.name = 'My location' AND b.name = 'Tokyo';

This will gives us:

	:::SQL
       st_distance    
    ------------------
    1602392.18109279

Meaning *1602.392 Kilometers*, a difference of about 45 Kilometers. That is indeed in between the Plate Carrée and our native geographical measurement.

Another, even more accurate and popular projection is our famous UTM. It can, however, not be used on a world scale. You can only perform measurements within the same UTM zone.

As mentioned in the previous chapter, there are roughly 60 World UTM zones on the earth, but each zone uses their own projection and their own coordinates.
This kind of projection is thus not fit for measuring distance on such a large scale.

Let us therefor take this one step further before I leave you to rest. Let us do a measurement with such a UTM projection.
We will make a measurement inside of Japan's mainland UTM zone: *54N* which has an SRID of *3095*.

First we will have to make another point in our database:

	:::SQL
	INSERT INTO shapes VALUES ('Aomori', ST_GeographyFromText('POINT(140.750616 40.788079)'));
	
This point represents the city of Aomori in northern Japan, famous for its huge lantern parades.

First let us measure with the native geographical calculations:

	:::SQL
	SELECT ST_Distance(a.location, b.location)
		FROM shapes a, shapes b 
		WHERE a.name = 'Aomori' AND b.name = 'Tokyo';

This returns:

	:::SQL
    st_distance    
    ------------------
    573416.203868172

Or *573.416 Kilometers*, which is most accurate.

Next, let us throw the good old *Plate Carrée* projection at it:

	:::SQL
	SELECT ST_Distance(a.location::geometry, b.location::geometry)
		FROM shapes a, shapes b 
		WHERE a.name = 'Aomori' AND b.name = 'Tokyo';

This will yield

	:::SQL
        st_distance    
    ------------------
    5.20224702126502

Which is in degrees again, doing this times 111 Kilometers will yield a total distance of *577.444 Kilometers*. 

Then let us measure using the correct UTM projection:

	:::SQL
	SELECT ST_Distance(ST_Transform(a.location::geometry, 3095), ST_Transform(b.location::geometry, 3095))
		FROM shapes a, shapes b 
		WHERE a.name = 'Aomori' AND b.name = 'Tokyo';

This will give us:

	:::SQL
      st_distance    
    ------------------
    573228.002047378

Or *573.228 Kilometers* and thus only around 200  meters different, in contrast with the Plate Carrée, which was 4 Kilometers different.

You can see that different projections will result in different measurements. It is therefor crucial to know which one to choose.
Some are better used on a local scale, like we just did for Japan, others are better on a global scale.

Again, it all comes down to trade-offs and choices.

Okay, yet another big chunk of PostGIS goodness is taken. I suggest a good rest of the mind.

We have seen how we can insert various types of geometry and geography, we saw how to manipulate and question them and we looked at a few real world measurements.

In the next and final chapter, we will be looking at loading some real GIS data from OpenStreetMap into our PostGIS database, take a quick look around my town here in Okinawa and take a deeper look at creating some important indexes.

And as always...thanks for reading!

<!--  LocalWords:  PostGIS PostgreSQL GIS
 -->
