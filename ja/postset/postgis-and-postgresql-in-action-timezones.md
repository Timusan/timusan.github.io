<!--
.. title: Postgis and PostgreSQL in Action - Timezones
.. slug: postgis-and-postgresql-in-action-timezones
.. date: 2014/08/20 10:00:00
.. tags: postgis, postgresql, timezone
-->

## Preface

Recently, I was lucky to be part of an *awesome* project called the [Breaking Boundaries Tour](http://breakingboundariestour.com).

This project is about two brothers, Omar and Greg Colin, who take their Stella scooters to make a full round trip across the United States.
And, while they are at it, try to raise funding for [Surfer's Healing Folly Beach](http://surfershealing.org/) - an organization that does great work enhancing the lives of children with autism through surfing .
To accommodate this trip, they wished to have a site where visitors could follow their trail *live*, as it happened.
A marker would travel across the map, with them, 24/7.

Furthermore, they needed the ability to jump off their scooters, snap a few pictures, edit a video, write some side info and push it on the net, for whole the world to see.
Immediately after they made their post, it had to appear on the exact spot they where at when snapping their moments of beauty.

To aid in the live tracking of their global position, they acquired a dedicated GPS tracking device which sends a latitude/longitude coordinate via a mobile data network every 5 minutes.

Now, this (short) post is not about how I build the entire application, but rather about how I used PostGIS and PostgreSQL for a rather peculiar matter: deducting timezone information.

For those who are interested though: the site is entirely build in Python using the Flask "micro framework" and, of course, PostgreSQL as the database.

## Timezone information?

Yes. Time, dates, timezones: hairy worms in hairy cans which many developers hate to open, but have to sooner or later.

In the case of Breaking Boundaries Tour, we had one major occasion where we needed the correct timezone information: where did the post happen?

## Where did it happen?

A feature we wanted to implement was one to help visitors get a better view of when a certain post was written.
To be able to see when a post was written in your local timezone is much more convenient then seeing the post time in some foreign zone.

We are lazy and do not wish to count back- or forward to figure out when a post popped up in our frame of time.

The reasoning is simple, always calculate all the times involved back to simple UTC (GMT). Then figure out the clients timezone using JavaScript, apply the time difference and done!

Simple eh?

Correct, except for one small detail in the feature request, in what zone was the post actually made?

Well...damn.

While you heart might be at the right place while thinking: "Simple, just look at the locale of the machine (laptop, mobile phone, ...) that was used to post!", this information if just too fragile. Remember, the bothers are *crossing* the USA, riding through at least three major timezones.
You can simply not expect all the devices involved when posting to always adjust their locale automatically depending on where they are.

We need a more robust solution. We need PostGIS.

But, how can a spatial database help us to figure out the timezone?

Well, thanks to the hard labor delivered to us by Eric Muller from [efele.net](http://efele.net), we have a *complete* and *maintained* shapefile of the entire world, containing polygons that represent the different timezones accompanied by the official timezone declarations.

This enables us to use the latitude and longitude information from the dedicated tracking device to pin point in which timezone they where while writing their post.

So let me take you on a short trip to show you how I used the above data in conjunction with PostGIS and PostgreSQL.

## Getting the data

The first thing to do, obviously, is to download the shapefile data and load it in to our PostgreSQL database.
Navigate to the [Timezone World](http://efele.net/maps/tz/world/) portion of the efele.net site and download the "tz_world" shapefile.

This will give you a zip which you can extract:

	:::bash
	$ unzip tz_world.zip
	
Unzipping will create a directory called "world" in which you can find the needed shapefile package files.

Next you will need to make sure that your database is PostGIS ready. Connect to your desired database (let us call it *bar*) *as a superuser*:

	:::bash
	$ psql -U postgres bar

And create the PostGIS extension:

	:::sql
	CREATE EXTENSION postgis;
	
Now go back to your terminal and load the shapefile into your database using the original owner of the database (here called *foo*):

	:::bash
	$ shp2pgsql -S -s 4326 -I tz_world | psql -U foo bar

As you might remember from the PostGIS series, this loads in the geometry from the shapefile using only simple geometry (not "MULTI..." types) with a SRID of 4326.

## What have we got?

This will take a couple of seconds and will create one table and two indexes. If you describe your database (assuming you have not made any tables yourself):

	:::bash
	public | geography_columns | view     | postgres
	public | geometry_columns  | view     | postgres
	public | raster_columns    | view     | postgres
	public | raster_overviews  | view     | postgres
	public | spatial_ref_sys   | table    | postgres
	public | tz_world          | table    | foo
	public | tz_world_gid_seq  | sequence | foo

You will see the standard PostGIS bookkeeping and you will find the *tz_world* table together with a *gid* sequence.

Let us describe the table:

	:::sql
	\d tz_world

And get:

	:::bash
	Column |          Type          |                       Modifiers                        
	--------+------------------------+--------------------------------------------------------
	gid    | integer                | not null default nextval('tz_world_gid_seq'::regclass)
	tzid   | character varying(30)  | 
	geom   | geometry(Polygon,4326) | 
	Indexes:
		"tz_world_pkey" PRIMARY KEY, btree (gid)
		"tz_world_geom_gist" gist (geom)

So we have:

* *gid*: an arbitrary id column
* *tzid*: holding the standards compliant textual timezone identification
* *geom*: holding polygons in *SRID* 4326.

Also notice we have two indexes made for us:

* *tz_world_pkey*: a simple B-tree index on our gid
* *tz_world_geom_gist*: a GiST index on our geometry

This is a rather nice set, would you not say?

## Using the data

So how do we go about using this data?

As I have said above, we need to figure out in which polygon (timezone) a certain point resides.

Let us take an arbitrary point on the earth:

* latitude: 35.362852
* longitude: 140.196131

This is a spot in the Chiba prefecture, central Japan.

Using the *Simple Features functions* we have available in PostGIS, it is trivial to find out in which polygon a certain point resides:

	:::sql
	SELECT tzid 
		FROM tz_world
		WHERE ST_Intersects(ST_GeomFromText('POINT(140.196131 35.362852)', 4326), geom);

And we get back:

	:::bash
		tzid    
	------------
	 Asia/Tokyo
	
*Awesome!*

In the above query I used the function *ST_Intersects* which checks if a given piece of geometry (our point) *shares any space* with another piece.
If we would check the execute plan of this query:

	:::sql
	EXPLAIN ANALYZE SELECT tzid 
		FROM tz_world
		WHERE ST_Intersects(ST_GeomFromText('POINT(140.196131 35.362852)', 4326), geom);
	
We get back:

	:::bash
	                                                      QUERY PLAN                                                          
	------------------------------------------------------------------------------------------------------------------------------
	Index Scan using tz_world_geom_gist on tz_world  (cost=0.28..8.54 rows=1 width=15) (actual time=0.591..0.592 rows=1 loops=1)
		Index Cond: ('0101000020E61000006BD784B446866140E3A430EF71AE4140'::geometry && geom)
		Filter: _st_intersects('0101000020E61000006BD784B446866140E3A430EF71AE4140'::geometry, geom)
	Total runtime: 0.617 ms

That is not bad at all, a runtime of little over 0.6 Milliseconds and it is using our GiST index.

But, if a lookup is using our GiST index, a small alarm bell should go off inside your head. Remember my last chapter on the PostGIS series?
I kept on babbling about index usage and how geometry functions or operators can only use GiST indexes when they perform *bounding box* calculations.

The latter might pose a problem in our case, for bounding boxes are a *very* rough approximations of the actual geometry.
This means that when we arrive near timezone borders, our calculations might just give us the wrong timezone.

So how can we fix this?

This time, we do not need to.

This is one of the few *blessed* functions that makes use of both an index *and* is very accurate.

The *ST_Intersects* first uses the index to perform bounding box calculations. This filters out the majority of available geometry.
Then it performs a more expensive, but more accurate calculation (on a small subset) to check if the given point is *really* inside the returned matches.

We can thus simply use this function without any more magic...life is simple!

## Implementation

Now it is fair to say that we do not wish to perform this calculation every time a user views a post, that would not be very efficient nor smart.

Rather, it is a good idea to generate this information at post time, and save it for later use.

The way I have setup to save this information is twofold:

* I only save a UTC (GTM) generalized timestamp of when the post was made.
* I made an extra column in my so-called "posts" table where I only save the string that represents the timezone (Asia/Tokyo in the above case).

This keeps the date/time information in the database naive of any timezone and makes for easier calculations to give the time in either the clients timezone or in the timezone the post was originally written.
You simply have one "root" time which you can move around timezones.

On every insert of a new post I have created a trigger that fetches the timezone and inserts it into the designated column.
You could also fetch the timezone and update the post record using Python, but opting for an in-database solution saves you a few extra, unneeded round trips and is most likely a lot faster.

Let us see how we could create such a trigger.

A trigger in PostgreSQL is an event you can set to fire when certain conditions are met. The event(s) that fire have to be encapsulated inside a PostgreSQL function.
Let us thus first start by creating the function that will insert our timezone string.
	
## Creating functions

In PostgreSQL you can write functions in either *C*, *Procedural* languages (PgSQL, Perl, Python) or plain *SQL*.

Creating functions with plain SQL is the most straightforward and most easy way. However, since we want to write a function that is to be used inside a trigger, we have even a better option.
We could employ the power of the embedded PostgreSQL procedural language to easily access and manipulate our newly insert data.

First, let us see which query we would use to fetch the timezone and update our post record:

	:::SQL
    UPDATE posts
	  SET tzid = timezone.tzid
	  FROM (SELECT tzid
		      FROM tz_world
			  WHERE ST_Intersects(
			    ST_SetSRID(
				  ST_MakePoint(140.196131, 35.362852),
				4326),
			  geom)) AS timezone
	  WHERE pid = 1;

This query will fetch the timezone string using a subquery and then update the correct record (a post with "pid" 1 in this example).

How do we pour this into a function?

	:::SQL
	CREATE OR REPLACE FUNCTION set_timezone() RETURNS TRIGGER AS $$
	BEGIN
        UPDATE posts
		SET tzid = timezone.tzid
		FROM (SELECT tzid 
			    FROM tz_world
				  WHERE ST_Intersects(
				    ST_SetSRID(
				      ST_MakePoint(NEW.longitude, NEW.latitude),
					4326),
				  geom)) AS timezone
		WHERE pid = NEW.pid;
		RETURN NEW;
	END $$
	LANGUAGE PLPGSQL IMMUTABLE;

First we use the syntax *CREATE OR REPLACE FUNCTION* to indicate we want to create (or replace) a custom function.
Then we tell PostgreSQL that this function will return type *TRIGGER*.

You might notice that we do not give this function any arguments. The reasoning here is that this function is "special".
Functions which are used as triggers magically get information about the inserted data available.

Inside the function you can see we access our latitude and longitude prefixed with *NEW*. These keywords, *NEW* and *OLD*, refer to the *record* after and before the trigger(s) happened.
In our case we could have used both, since we do not alter the latitude or longitude data, we simply fill a column that is NULL by default.
There are more keywords available (*TG_NAME*, *TG_RELID*, *TG_NARGS*, ...) which refer to properties of the trigger itself, but that is beyond today's scope.

The actual SQL statement is wrapped between double dollar signs (*$$*). This is called *dollar quoting* and is the preferred way to quote your SQL string (as opposed to using single quotes).
The body of the function, which in our case is mostly the SQL statement, is surrounded with a *BEGIN* and *END* keyword.

A trigger function always needs a *RETURN* statement that is used to provide the data for the updated record. This too has to reside in the body of the function.

Near the end of our function we need to declare in which language this function was written, in our case *PLPGSQL*.

Finally, the *IMMUTABLE* keyword tells PostgreSQL that this function is rather "functional", meaning: if the inputs are the same, the output will also, *always* be the same.
Using this *caching* keyword gives our famous PostgreSQL planner the ability to make decisions based on this knowledge.

## Creating triggers

Now that we have this functionality wrapped into a tiny PLPGSQL function, we can go ahead and create the trigger.

First you have the event on which a trigger can execute, these are:

* INSERT
* UPDATE
* DELETE
* TRUNCATE

Next, for each event you can specify at what timing your trigger has to fire:

* BEFORE
* AFTER
* INSTEAD OF

The last one is a special timing by which you can replace the default behavior of the mentioned events.

For our use case, we are interested in executing our function *AFTER INSERT*.

	:::sql
	CREATE TRIGGER set_timezone
		AFTER INSERT ON posts
		FOR EACH ROW
		EXECUTE PROCEDURE set_timezone();

This will setup the trigger that fires after the insert of a new record.

## Wrapping it up

Good, that all there is to it.

We use a query, wrapped in a function, triggered by an insert event to inject the official timezone string which is deducted by PostGIS's spatial abilities.

Now you can use this information to get the exact timezone of where the post was made and use this to present the surfing client both the post timezone time and their local time.

For the curious ones out there: I used the [MomentJS](http://momentjs.com/ MomentJS JavaScript library) library for the client side time parsing. This library offers a timezone extension which accepts these official timezone strings to calculate offsets. A lifesaver, so go check it out.

Also, be sure to follow the bros while they scooter across the States!

And as always...thanks for reading!
