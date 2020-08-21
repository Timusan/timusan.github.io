<!--
.. title: Postgis, PostgreSQL's spatial partner - Part 3
.. slug: postgis-postgresqls-spatial-partner-part-3
.. date: 2014/06/25 10:00:00
.. tags: postgis, postgresql
-->

You have arrived at the final chapter of this PostGIS introduction  series. Before continuing, I recommend you read [chapter one](http://shisaa.jp/postset/postgis-postgresqls-spatial-partner-part-1.html "Part one of this series.") and [chapter two](http://shisaa.jp/postset/postgis-postgresqls-spatial-partner-part-2.html "Part one of this series.") first.

In the last chapter we finished by doing some real world distance measuring and we saw how different projections pushed forward different results.

Today I would like to take this practical approach a bit further and continue our work with real world data by showing you around the town of Kin in Okinawa. The town where I live.

## A word before we start

In this chapter I want to do a few experiments together with you on real world data.
To gather this data, I would like to use OpenStreetMap because it is not only *open* but also gives us handy tools to export map information.

We will use a tool called *osm2pgsql* to load our OSM data into PostGIS enable tables.

However, it is more common to import and export real world GIS data by using the semi-closed ESRI standard *shapefile* format.
OpenStreetMap does not support exporting to this shapefile format directly, but exports to a more open XML file (.osm) instead.

Therefor, near the end of this post, we will briefly cover these shapefiles as well and see how we could import them into our PostgreSQL database.
But for the majority of our work today, I will focus on the OpenStreetMap approach.

## The preparation

Let us commence with this adventure by first getting all the GIS data related to the whole of Okinawa.
We will only be interested in the data related to Kin town, but I need you to pull in a data set that is large enough (but still tiny in PostgreSQL terms) for us to experiment with indexing.

Hop online and download the file being served at the following URL: [openstreetmap.org Okinawa island](http://overpass-api.de/api/map?bbox=126.079,25.596,130.852,28.898)
It is a file of roughly 180 Mb and covers most of the Okinawan main island. Save the presented "map" file.

Next we will need to install a third party tool which is specifically designed to import this OSM file into PostGIS.
This tool is called *osm2pgsql* and is available in many Linux distributions.

On a Debian system:

	:::bash
	apt-get install osm2pgsql

## Loading foreign data

Now we are ready to load in this data. But first, let us clean our "gis" database we used before.

Since all these import tools will create their own PostGIS enabled tables, we can delete our "shapes" table. Connect to your "gis" database and drop this table:

	:::sql
	DROP TABLE shapes;

Using this new tool, repopulate the "gis" database with the data you just downloaded:

	:::bash
	osm2pgsql -s -U postgres -d gis map

If everything went okay, you will get a small report containing the information about all the tables *and* indexes that where created.

Let us see what we just did. 

First we ran *osm2pgsql* with the *-s* flag. This flag enabled *slim* mode, which means it will use a database on disk, rather then processing all the GIS data in RAM.
The latter does not only potentially slow down your machine for larger data sets, but it enables less features to be available.

Next we tell the tool to connect as the user "postgres" and load the data into the "gis" database. The final argument is the "map" file you just downloaded.

## What do we have now?

Open up a database console and let us describe our database to see what this tool just did:

	:::sql
	\d
	
As you can see, it inserted 7 new tables:

	:::bash
	Schema |        Name        | Type  |  Owner   
	--------+--------------------+-------+----------
	public | geography_columns  | view  | postgres
	public | geometry_columns   | view  | postgres
	public | planet_osm_line    | table | postgres
	public | planet_osm_nodes   | table | postgres
	public | planet_osm_point   | table | postgres
	public | planet_osm_polygon | table | postgres
	public | planet_osm_rels    | table | postgres
	public | planet_osm_roads   | table | postgres
	public | planet_osm_ways    | table | postgres
	public | raster_columns     | view  | postgres
	public | raster_overviews   | view  | postgres
	public | spatial_ref_sys    | table | postgres

The other 5 views and tables are the good old PostGIS bookkeeping.

It is also important, yet less relevant for our work here today, to know that these tables, or rather the way *osm2pgsql* imports, is optimized to work with *Mapnik*.
Mapnik is an open-source map rendering software package used for both web and offline usage.

The tables that are imported contain many different types of information. Let me quickly go over them to give you a basic feeling of how the import happened:

* planet_osm_line: holds all non-closed pieces of geometry (called *ways*) at a high resolution. They mostly represent actual roads and are used when looking at a small, zoomed-in detail of a map.
* planet_osm_nodes: an intermediate table that holds the raw point data (points in lat/long) with a corresponding "osm_id" to map them to other tables
* planet_osm_point: holds all points-of-interest together with their OSM tags - tags that describe what they represent
* planet_osm_polygon: holds all closed piece of geometry (also called *ways*) like buildings, parks, lakes, areas, ...
* planet_osm_rels: an intermediate table that holds extra connecting information about polygons
* planet_osm_roads: holds lower resolution, non-closed piece of geometry in contrast with "planet_osm_line". This data is used when looking at a greater distance, covering much area and thus not much detail about smaller, local roads.
* planet_osm_ways: an intermediate table which holds non-closed geometry in raw format

We will now continue working with a small subset of this data.

Let us take a peek at the Polygons tables for example. First, let us see what we have available:

	:::sql
	\d planet_osm_polygon
	
That is quite a big list, but the major part of these columns are of mere TEXT type and contain human information about the geometry stored.
These columns corresponds with the way OpenStreetMap categorizes their data and with the way you could use the Mapnik software described above.

Let us do a targeted query:

	:::sql
	SELECT name, building, ST_AsText(way)
		FROM planet_osm_polygon
		WHERE building = 'industrial';

Notice that I use the *output* function *ST_AsText()* to convert to a human readable WKT string.
Also, I am only interested in some of the industrial buildings, so I set the building type to *industrial*.

The result:

	:::bash
                        name                      |  building  |                                                                     st_astext                                                                   
    ----------------------------------------------+------------+---------------------------------------------------------------------------------------------------------------------------------------------------------------
     沖ハム (Okiham)                               | industrial | POLYGON((14221927.83 3049797.01,14222009.77 3049839.68,14222074.84 3049714.68,14222028.9 3049690.76,14221996.33 3049753.33,14221960.34 3049734.58,14221927.83 3049797.01))
	Kin Thermal Power Plant Coal storage building | industrial | POLYGON((14239931.42 3054117.72,14239990.49 3054224.25,14240230.15 3054091.38,14240171.08 3053984.84,14239931.42 3054117.72))
	Kin Thermal Power Plant Exhaust tower         | industrial | POLYGON((14240167.1 3054497.14,14240172.26 3054507.93,14240176.04 3054515.82,14240195.76 3054506.39,14240186.84 3054487.7,14240167.1 3054497.14))


We get back three records containing one industrial building each, described with a closed polygon. Cool.

Now, I can assure you that Okinawa has more then three industrial buildings, but do remember that we are looking at a rather rural island.
OpenStreetMap relies greatly on user generated content and there simply are not many users who have felt the need to index the industrial buildings here in this neck of the woods.

The *planet_osm_polygon* table does contain little over 6000 buildings of various types, which is still a small number, but for our purpose today I am only interested in the latter two, which both lie here in Kin town.

Also, if you would, for example, take a chunk of Tokyo, where there are hundreds of active OpenStreetMap contributors, you will find that many buildings are present and are sometimes even more accurately represented then some other online proprietary mapping solutions offered by some famous search engines. Ahum.

Before continuing, though, I would like to delete two GiST indexes that "osm2pgsql" made for us, purely to be able to demonstrate the importance of an index.

For now, just take my word and delete the indexes on all the geometry columns of the tables we will use today:

	:::sql
	DROP INDEX planet_osm_line_index;
	DROP INDEX planet_osm_polygon_index;

Then perform a VACUUM:

	:::sql
	VACUUM ANALYZE planet_osm_line;
	VACUUM ANALYZE planet_osm_polygon;
	
*VACUUM* together with *ANALYZE* will force PostgreSQL to recheck the whole table for any changed conditions, as is the case since we removed the index.

The first thing I would like to find out is how large these building actually are.
We cannot measure how tall they are, for we are working with two dimensional data here, but we can measure their footprint on the map.

Since PostGIS makes all of our work easy, we could simply employ a function to tell us this information:

	:::sql
	SELECT ST_Area(way)
		FROM planet_osm_polygon
		WHERE building = 'industrial';

And get back:

	:::bash
		st_area      
	------------------
	10155.3935499731
	33381.1043500491
	452.9464999972

As we know from the previous chapter, to be able to know what these numbers mean, we have to find out in which SRID this data was saved.
You could either describe the table again and look at the geometry column description, or use an *accessor* function *ST_SRID()*, to find it:

	:::sql
	SELECT ST_SRID(way)
		FROM planet_osm_polygon;
		WHERE building = 'industrial';

And get back:

	:::sql
	 st_srid 
	---------
	900913
	900913
	900913
	
You could also query the PostGIS bookkeeping directly and look in the *geometry_columns* view:

	:::sql
	SELECT f_tablename, f_geometry_column, coord_dimension, srid, type
		FROM geometry_columns;

This view holds information about all the geometry columns in our PostGIS enabled database.
Our above query will return a list containing all the GIS describing information we saw in the previous chapter.

Nice. Both our buildings are stored in a geometry column and have an SRID of *900913*. We can now use our *spatial_ref_sys* table to look up this ID:

	:::sql
	SELECT srid, auth_name, auth_srid, srtext, proj4text
		FROM spatial_ref_sys
		WHERE srid = 900913;
		
As you can see, this is basically a Mercator projection used by OpenStreetMap.
In the "proj4text" column we can see that its units are meters.

This thus means that the information we get back is in *square Meters*.

In this map (only looking at the latter two Kin buildings) we thus have a building with a total area of 33 *square Kilometers* and a more modest building of around 452 *square Meters*.
The former is a coal storage facility belonging to the *Kin Thermal Power Plant* and is indeed *huge*.
The second building represents the exhaust tower of that same plant.

You have just measured the area these buildings occupy, very neat right?

Now, let us find out which road runs next to this power plant, just in case we wish to drive to there.
It is important to note that OSM (and many other mapping solutions) divide roads into different types.

You have trunk roads, highways, secondary roads, tertiary roads, etc.
I am now interested to find the nearest *secondary* road.

To get a list of all the secondary roads in Okinawa, simply query the *planet_osm_roads* table:

	:::sql
	SELECT ref, ST_AsText(way) 
		FROM planet_osm_line
		WHERE highway = 'secondary';

We now get back all the linestring objects together with their reference inside of OSM.
The reference refers to the actual route number each road has.

The total count should be around *3215* pieces of geometry, which is already a nice list to work with.

Let us now see which of these roads is closest to our coal storage building.

To find out how far something is (nearest neighbor search) we could use our *ST_Distance()* function we used in the previous chapter and perform the following lookup:

	:::sql
	SELECT road.highway, road.ref, ST_Distance(road.way, building.way) AS distance
		FROM planet_osm_polygon AS building, planet_osm_line AS road
		WHERE road.highway = 'secondary' AND building.name = 'Kin Thermal Power Plant Coal storage building'
		ORDER BY distance;

This will bring us:

	:::bash
	 highway  | ref |     distance     
	-----------+-----+------------------
	secondary | 329 | 417.374986575458
	secondary | 104 | 2258.90394593648
	secondary | 104 | 2709.00178089638
	secondary | 104 | 2745.76782385198
	secondary | 234 | 5897.78205314507
	...

Cool, secondary route 329 is the closest to our coal storage building with a distance of *417 meters.*

While this will return quite accurate results, there is one problem with this query. Indexes are not being used.
And every time an index is potentially left alone, you should start to worry, especially with larger data sets.

How do I know they are ignored? Simple, we did not make any indexes (and we deleted the ones made by "osm2pgsql")...which makes me pretty sure we cannot use them.

I refer you to [chapter three](http://shisaa.jp/postset/postgresql-full-text-search-part-3.html) of my PostgreSQL Full Text series where I talk a bit more about GiST and B-Tree index types.
And, as I also say in that chapter, I highly recommend reading Markus Winand's [Use The Index, Luke](http://use-the-index-luke.com/ "Use The Index, Luke series written by Markus Winand.") series, which explains in great detail how database indexes work.

The first thing to realize is that an index will only be used if the data set on which it is build is of sufficient size.
PostgreSQL has an AI build in, called the *query planner*, which will make a decision on whether or not to use an index.

If your data set is small enough a more traditional *Sequential Scan* will be faster or equal.

To know what is going on *exactly* and to know *how fast* our query runs, we have the *EXPLAIN* command at our disposal.

## Speeding things up

Let us *EXPLAIN* the query we have just run:

	:::sql
	EXPLAIN ANALYZE SELECT road.highway, road.ref, ST_Distance(road.way, building.way) AS distance
		FROM planet_osm_polygon AS building, planet_osm_line AS road
		WHERE road.highway = 'secondary' AND building.name = 'Kin Thermal Power Plant Coal storage building'
		ORDER BY distance;

We simply put the keyword *EXPLAIN* (and *ANALYZE* to give us total runtime) right in front of our normal query.

The result:

	:::bash
	Sort  (cost=5047.50..5055.32 rows=3129 width=391) (actual time=41.481..41.815 rows=3215 loops=1)
		Sort Key: (st_distance(road.way, building.way))
		Sort Method: quicksort  Memory: 348kB
		->  Nested Loop  (cost=0.00..4309.34 rows=3129 width=391) (actual time=1.188..38.617 rows=3215 loops=1)
			->  Seq Scan on planet_osm_polygon building  (cost=0.00..279.01 rows=1 width=207) (actual time=0.981..1.409 rows=1 loops=1)
				Filter: (name = 'Kin Thermal Power Plant Coal storage building'::text)
				Rows Removed by Filter: 6320
			->  Seq Scan on planet_osm_line road  (cost=0.00..3216.79 rows=3129 width=184) (actual time=0.166..26.524 rows=3215 loops=1)
				Filter: (highway = 'secondary'::text)
				Rows Removed by Filter: 73488
    Total runtime: 42.153 ms

That is a lot of output, but it shows you how the internal planner executes our query and which decisions it makes along the way.

To fully interpret a query plan (this is still a simple one), a lot more knowledge is needed and this would easily deserve its own *series*.
I am by far not an expert in the query planner (though it is an interesting study topic), but I will do my best to extract the important bits we need for our direct performance tuning.

A query plan is always made up out of nested nodes, the parent node containing all the accumulated information (costs, rows, ...) of its child nodes.

Inside the nested loop parent node we see above, we can find that the planner decided to use two filters, which correspond to the *WHERE* clause conditions of our query (building.name and road.highway).
You can see that both child nodes are of *Seq Scan* type, which means *Sequential Scan*. These types of nodes scan the whole table, simply from top to bottom, directly from disk.

Another important thing to note is the total time this query costs, which is *42.153 ms*.
The time reported here is the time on my local machine, depending on how decent your computer is, this time could vary.

A detail not to forget when looking at this timing, is the fact that it is slightly skewed if compared to real-world application use:

* We neglect network/client traffic. This query now runs internally and does not need to communicate with a client driver (which almost always brings extra overhead)
* The time measurement itself also introduces overhead.

The total runtime from our above plan does not sound as a big number, but we are working with a rather small data set - the area of Okinawa is large, but the geometry is rather sparse.

So our first reaction should be: this can be better.

First, let us try to get rid of these sequential scans, for they are a clear indication that the planner does not use an index.

### Creating indexes

In our case we want to make two types of indexes:

* Indexes on our "meta" data, the names and other attributes describing out geometrical data
* Indexes that actually index our geometrical data itself

Let us start with our attributes columns.

These are all simple VARCHAR, TEXT or INT columns, so the good old Balanced Tree or *B-Tree* can be used here.
In our query above we use "road.highway" and "building.name" in our lookup, so let us make a couple of indexes that adhere to this query.
Remember, an index only makes sense if it is built the same way your queries question your data.

First, the "highway" column of the "planet_osm_line" table:

	:::sql
	CREATE INDEX planet_osm_line_highway_index ON planet_osm_line(highway);

The syntax is trivial. You simply tell PostgreSQL to create an index, give it a name, and tell it on which column(s) of which table you want it to be built.
PostgreSQL will always default to the *B-Tree* index type.

Next, the name column:

	:::sql
	CREATE INDEX planet_osm_polygon_name_index ON planet_osm_polygon(name);

Now perform another *VACUUM ANALYZE* on both tables:

	:::sql
	VACUUM ANALYZE planet_osm_line;
	VACUUM ANALYZE planet_osm_polygon;

Let us run explain again on the exact same query:

	:::sql
	Sort  (cost=4058.73..4066.56 rows=3129 width=394) (actual time=20.817..21.149 rows=3215 loops=1)
		Sort Key: (st_distance(road.way, building.way))
		Sort Method: quicksort  Memory: 348kB
		->  Nested Loop  (cost=72.95..3310.07 rows=3129 width=394) (actual time=1.356..17.743 rows=3215 loops=1)
			->  Index Scan using planet_osm_polygon_name_index on planet_osm_polygon building  (cost=0.28..8.30 rows=1 width=207) (actual time=0.054..0.056 rows=1 loops=1)
				Index Cond: (name = 'Kin Thermal Power Plant Coal storage building'::text)
            ->  Bitmap Heap Scan on planet_osm_line road  (cost=72.67..2488.23 rows=3129 width=187) (actual time=1.258..4.661 rows=3215 loops=1)
		        Recheck Cond: (highway = 'secondary'::text)
				->  Bitmap Index Scan on planet_osm_line_highway_index  (cost=0.00..71.89 rows=3129 width=0) (actual time=0.864..0.864 rows=3215 loops=1)
                       Index Cond: (highway = 'secondary'::text)
    Total runtime: 21.527 ms
	
You can see that we now traded our *Seq Scan* for *Index Scan* and *Bitmap Heap Scan*, which indicates that our attribute indexes are being used, yay!

The so-called *Bitmap Heap Scan*, instead of a *Sequential Scan*, is performed when the planner decides it can use the index to gather all the rows it thinks it needs, sort them in logical order and then fetch the data from the table on disk in the most optimized way possible (trying to open each disk page only once).

The order by which the *Bitmap Heap Scan* arranges the data is directed by the child node aka the *Bitmap Index Scan*. This latter type of node is the one doing the actual searching *inside* the index. Because in our *WHERE* clause we have a condition which tells PostgreSQL to limit the rows to the ones of "highway" type "secondary", the *Bitmap Index Scan* fetches the needed rows from our *B-Tree* index we just made and passes them to its parent, the *Bitmap Heap Scan*, which then goes on to order the geometry rows to be fetched.

This already helped much, for our query runtime dropped to half. Now, let us make the indexes for our actual geometry, and see the effect:

	:::sql
	CREATE INDEX planet_osm_line_way ON planet_osm_line USING gist(way);
	CREATE INDEX planet_osm_polygon_way ON planet_osm_polygon USING gist(way);

Creating a *GiST* index is quite similar to a normal *B-Tree* index. The only difference here is that you specify the index to be build with *GiST*.

Vacuum:

	:::sql
	VACUUM ANALYZE planet_osm_line;
	VACUUM ANALYZE planet_osm_polygon;

Now poke it again with the same query and see our new plan:

	:::bash
	Sort  (cost=4038.82..4046.54 rows=3089 width=395) (actual time=21.137..21.479 rows=3215 loops=1)
		Sort Key: (st_distance(road.way, building.way))
		Sort Method: quicksort  Memory: 348kB
		->  Nested Loop  (cost=72.64..3299.76 rows=3089 width=395) (actual time=1.382..17.858 rows=3215 loops=1)
			->  Index Scan using planet_osm_polygon_name_index on planet_osm_polygon building  (cost=0.28..8.30 rows=1 width=207) (actual time=0.041..0.044 rows=1 loops=1)
                Index Cond: (name = 'Kin Thermal Power Plant Coal storage building'::text)
			->  Bitmap Heap Scan on planet_osm_line road  (cost=72.36..2488.32 rows=3089 width=188) (actual time=1.297..4.726 rows=3215 loops=1)
				Recheck Cond: (highway = 'secondary'::text)
				->  Bitmap Index Scan on planet_osm_line_highway_index  (cost=0.00..71.59 rows=3089 width=0) (actual time=0.866..0.866 rows=3215 loops=1)
                      Index Cond: (highway = 'secondary'::text)
    Total runtime: 21.873 ms

Hmm, the plan did not change at all, and our runtime is roughly identical. Why is our performance still the same?

The culprit here is *ST_Distance()*.

As it turns out, this function is unable to use the *GiST* index and is therefor not a good candidate to set loose on your whole result set. The same goes for the *ST_Area()* function, by the way.

So we need a way to limit the amount of records we do this expensive calculation on.

### ST_DWithin()

We introduce a new function: *ST_DWithin()*. This function could be our savior in this case, for it does use the *GiST* index.

Whether or not a function (or operator) can use the *GiST* index, depends on if it uses *bounding boxes* when performing calculations.
The reason why is because *GiST* indexes mainly store bounding box information and not the exact geometry itself.

*ST_DWithin()* checks if given geometry is within a radius of another piece of geometry and simply returns *TRUE* or *FALSE*.
We can thus use it in our *WHERE* clause to filter out geometry for which it returns *FALSE* (and thus not falls within the radius).
It performs this check using bounding boxes, and thus is able to retrieve this information from our *GiST* index.

Let me present you with a query that limits the result set based on what *ST_DWithin()* finds:

	:::sql
	EXPLAIN ANALYZE SELECT road.highway, road.ref, ST_Distance(road.way, building.way) AS distance
		FROM planet_osm_polygon AS building, planet_osm_line AS road
		WHERE road.highway = 'secondary'
			AND building.name = 'Kin Thermal Power Plant Coal storage building'
			AND ST_DWithin(road.way, building.way, 10000)
		ORDER BY distance;

As you can see we simply added one more *WHERE* clause to limit the returned geometry by radius.
This will result in the following plan:
	
	:::bash
	Sort  (cost=45.66..45.67 rows=1 width=395) (actual time=6.048..6.052 rows=27 loops=1)
		Sort Key: (st_distance(road.way, building.way))
		Sort Method: quicksort  Memory: 27kB
		->  Nested Loop  (cost=4.63..45.65 rows=1 width=395) (actual time=3.157..6.005 rows=27 loops=1)
			->  Index Scan using planet_osm_polygon_name_index on planet_osm_polygon building  (cost=0.28..8.30 rows=1 width=207) (actual time=0.051..0.054 rows=1 loops=1)
				Index Cond: (name = 'Kin Thermal Power Plant Coal storage building'::text)
			->  Bitmap Heap Scan on planet_osm_line road  (cost=4.34..37.09 rows=1 width=188) (actual time=3.090..5.771 rows=27 loops=1)
				Recheck Cond: (way && st_expand(building.way, 10000::double precision))
				Filter: ((highway = 'secondary'::text) AND (building.way && st_expand(way, 10000::double precision)) AND _st_dwithin(way, building.way, 10000::double precision))
				Rows Removed by Filter: 4838
					->  Bitmap Index Scan on planet_osm_line_way  (cost=0.00..4.34 rows=8 width=0) (actual time=1.978..1.978 rows=4865 loops=1)
						Index Cond: (way && st_expand(building.way, 10000::double precision))
    Total runtime: 6.181 ms

Good. We have just gone down to only *6.181 ms*, That seems to be much more efficient.

As you can see, our query plan got a few new rows. The main thing to notice is the fact that our *Bitmap Heap Scan* got another *Recheck Cond*, our expanded *ST_DWithin()* condition.
More to the bottom, you can see that the condition is being pulled from the *GiST* index:

	:::bash
	Index Cond: (way && st_expand(building.way, 10000::double precision))

This seems to be a much more desirable and scalable query.

But there is a drawback, though *ST_DWithin()* will make for speedy results, it works only by giving it a fixed radius.

As you can see from our usage, we call the function as follows: ST_DWithin(road.way, building.way, 10000).
The last argument, "10000", tells us how big the search radius is. In this case our geometry is in meters, so this means we search in a radius of 10 Km.

This static radius number is quite arbitrary and might not always be desirable. What other options do we have without compromising performance too much?

### Operators

Another addition of PostGIS we have not talked about much up until now are the spatial *operators* we have available.
You have a total of 16 operators you can use to perform matches on your GIS data.

You have straightforward operators like *&&*, which returns *TRUE* if one piece of geometry intersects with another (bounding box calculation) or the *<<* which returns *TRUE* if one object is fully to the left of another object.

But there are more interesting ones like the *<->* and the *<#>* operators.

The first operator, *<->*, returns the distance between two points. If you feed it other types of geometry (like a linestring of polygon) it will first draw a bounding box around that geometry and perform a point calculation by using the bounding box *centroids*. A centroid is the calculated center of a piece of geometry (the drawn bounding box in our case).

The second, *<#>*, acts completely the same, but works directly on bounding boxes of given geometry. In our case, since we are not working with points, it would make more sense to use this operator.

The big advantage of this distance calculation operator is, once more, the fact that it too calculates using a bounding box and is thus able to use a *GiST* index.
However, the *ST_Distance()* function calculates distances by finding two points on the given geometry most close to each other, which serves the most *accurate* result.
The *<#>* operator, as said before, stretches a *bounding box* around each piece of geometry and therefor deforms our objects, making for less accurate distance measuring.

It is therefor not wise to use *<#>* to calculate accurate distances, but it is a life saver to *sort away* geometry that is too far away for our interest.

So a proper usage would be to first *roughly* limit the result set using the *<#>* operator and then more accurately measure the distance of, say, the first 50 matches with our famous *ST_Distance()*.

Before we can continue, it is important to point out that both the *<->* and *<#>* operator can only use the *GiST* index when either the left or right hand side of the operator is a *constant* or *fixed* piece of geometry. This means we have to provide actual geometry using a constructor function.

There are other ways around this limitation by, for example as Alexandre Neto points out on the PostGIS mailing list, providing your own function which converts our "dynamic" geometry into a constant.

But this would make this post run way past its initial focus.
Let us simply try by providing a fixed piece of geometry.
The fixed piece is, of course, still our "Kin Thermal Power Plant Coal storage building", but converted into WKT:

	:::sql
	EXPLAIN ANALYZE WITH distance AS (
		SELECT way AS road, ref AS route
			FROM planet_osm_line
			WHERE highway = 'secondary'
			ORDER BY ST_GeomFromText('POLYGON((14239931.42 3054117.72,14239990.49 3054224.25,14240230.15 3054091.38,14240171.08 3053984.84,14239931.42 3054117.72))', 900913) <#> way
			LIMIT 50
			)
    SELECT ST_Distance(ST_GeomFromText('POLYGON((14239931.42 3054117.72,14239990.49 3054224.25,14240230.15 3054091.38,14240171.08 3053984.84,14239931.42 3054117.72))', 900913), road) AS true_distance, route
		FROM distance
		ORDER BY true_distance
		LIMIT 1;

This query uses a *Common Table Expression* or *CTE* (you could also use a simpler subquery) to first get a rough result set of about 50 rows based on what *<#>* finds.
Then *only* on those 50 rows do we perform our more expensive, index-agnostic distance calculation.

This results in the following plan and runtime:


	:::bash
	Limit  (cost=274.57..274.57 rows=1 width=64) (actual time=11.236..11.237 rows=1 loops=1)
		CTE distance
			->  Limit  (cost=0.28..260.82 rows=50 width=173) (actual time=0.389..10.764 rows=50 loops=1)
				->  Index Scan using planet_osm_line_way on planet_osm_line  (cost=0.28..16362.19 rows=3140 width=173) (actual time=0.389..10.745 rows=50 loops=1)
					Order By: (way <#> '010300002031BF0D000100000005000000D7A3706D17296B41C3F528DC124D47417B14AECF1E296B4100000020484D4741CDCCCCC43C296B410AD7A3B0054D4741295C8F6235296B41B81E856BD04C4741D7A3706D17296B41C3F528DC124D4741'::geometry)
					Filter: (highway = 'secondary'::text)
					Rows Removed by Filter: 4562
				->  Sort  (cost=13.75..13.88 rows=50 width=64) (actual time=11.234..11.234 rows=1 loops=1)
					Sort Key: (st_distance('010300002031BF0D000100000005000000D7A3706D17296B41C3F528DC124D47417B14AECF1E296B4100000020484D4741CDCCCCC43C296B410AD7A3B0054D4741295C8F6235296B41B81E856BD04C4741D7A3706D17296B41C3F528DC124D4741'::geometry, distance.road))
					Sort Method: top-N heapsort  Memory: 25kB
        ->  CTE Scan on distance  (cost=0.00..13.50 rows=50 width=64) (actual time=0.412..11.188 rows=50 loops=1)
    Total runtime: 11.268 ms
	
As you can see, we are now using the *GiST* index "planet_osm_line_way", which was what we were after.

This yields roughly the same runtime as with our *ST_DWithin()*, but without the arbitrary distance setting.
We indeed have a somewhat arbitrary limiter of 50, but this is much less severe then a distance limiter.

Even if the closest secondary road is 100 Km from our building, the above query would still find it whereas our previous query would return nothing.

## One more for the road home

Let us do a few more fun calculations on our Okinawa data, before I let you off the island.

Next I would like to find the longest *trunk* road that runs through this prefecture:

	:::sql
	SELECT ref, highway, ST_Length(way) as length
		FROM planet_osm_line 
		WHERE highway = 'trunk'
		ORDER BY length DESC;
		
We have a new function *ST_Length()* which simply returns the length, given that the geometry is a linestring or multilinestring.
The only index that will be used is our "planet_osm_line_highway_index" *B-Tree* index to perform our *Bitmap Index Scan*.

*ST_Length()* does obviously not work with bounding boxes and therefor cannot use the geometrical *GiST* index. This is yet another function you should use carefully.

When looking at the result set that was returned to us, you will see that some routes show up multiple times.
Take route *58*, which is the longest and most famous route in Okinawa. It shows up around *769* times. Why?

This is because, especially for a database prepared for mapping, these pieces of geometry are divided over different tiles.

We thus need to accumulate the length of all the linestrings we find that represent pieces of route 58.
First, we could try to accomplish this with plain SQL:

	:::sql
	WITH road_pieces AS (
		SELECT ST_Length(way) AS length
		FROM planet_osm_line
		WHERE ref = '58'
		)
	SELECT sum(length) AS total_length
	FROM road_pieces;
	
This will return:

	:::bash
	536468.804010367

Meaning a total length of *536.486 Kilometers*. This query will run in about *19.375 ms*.
Let us add an index to our "ref" column:

	:::sql
	CREATE INDEX planet_osm_line_ref_index ON planet_osm_line(ref);

Perform vacuum:

	:::sql
	VACUUM ANALYZE planet_osm_line;

This index creation will speed up to query and make it run in little over *3.524 ms*. Nice runtime.

You could also perform almost the exact same query, but instead of using an SQL sum() function, you could use *ST_Collect()*, which creates collections of geometry out of all the separate pieces you feed it.
In our case we feed it separate linestrings, which will make this function output a single *multilinestring*. We would then only have to perform one length calculation.

	:::sql
	WITH road_pieces AS (
		SELECT ST_Collect(way) AS geom
		FROM planet_osm_line
		WHERE ref = '58'	
		)
	SELECT ST_Length(geom) AS length
		FROM road_pieces;

This query will run even around *1 ms* faster then former and it returns *the exact* same distance of *536.486 Kilometers*.

Now that we have this one multilinestring which represents route 58, we could check how close this route comes to our famous Kin building (which we will statically feed):

	:::sql
	WITH road_pieces AS (
		SELECT ST_Collect(way) AS geom
		FROM planet_osm_line
		WHERE ref = '58'	
		)
	SELECT ST_Distance(geom, ST_GeomFromText('POLYGON((14239931.42 3054117.72,14239990.49 3054224.25,14240230.15 3054091.38,14240171.08 3053984.84,14239931.42 3054117.72))',900913)) AS distance
		FROM road_pieces;

Which would give us:

	:::bash
	 7900.58662432767

In other words: Route 58 is, at it closest point, *7.9 Kilometers* from our coal storage building.
This query now took about *5 ms* to complete. A rather nice throughput.

Okay, enough exploring for today.

We took a brief look at indexing our spatial data, and what benefits we could gain from it.
And, as you can imagine, a lack of indexes and improper use of the GIS functions, could lead to dramatic slow-downs, certainly on larger data sets.

## Shapefiles

Before I will let you go I want to take a brief look at another mechanism of carrying around GIS data: the *shapefile*.
Probably more used then the OSM XML format, but less open. It is almost the GIS standard way of exchanging data between GIS systems.

We can import shapefiles by using a tool called "shp2pgsql" which comes shipped with PostGIS.
This tool will attempt to upload *ESRI* shape data into your PostGIS enables database.

### ESRI?

*ESRI* stands for *Environmental Systems Research Institute* and is yet another organization that taps into the world of digital cartography.

They have defined a (somewhat open) file format standard that allows the GIS world to save their data in a so called *shapefile*.
These files hold GIS primitives (polygons, linestrings, points, ...) together with a bunch of descriptive information that tells us what each primitive represents.

It was once developed for ESRI's own, proprietary software package (ArcGIS), but was quickly picked up by the rest of the GIS community.
Today, almost all serious GIS packages have the ability to read and/or write to such shapefiles.

### Shapefile build-up

Let us take a peek at the guts of such a shapefile.

First, contrary to what the name suggest, a shapefile is not a single file. At a minimal level, it is a bundle containing a minimum of three files to be spec compliant:

* *.shp*: the first mandatory file has the extension *.shp* and holds the GIS primitives themselves.
* *.shx*: the second important file is an index of the geometry 
* *.dbf*: the last needed file is a database file with geometry attributes

## Getting shapefile data

There are many organizations who offer shapefiles of all areas of the globe, either free or for a small fee.
But since we already have data in our database we are familiar with, we could create our own shapefiles.

### Exporting with pgsql2shp

Besides "shp2pgsql", which is used to import or *load* shapefiles, we also got shipped a reverse tool called "pgsql2shp", which can export to or *dump* shapefiles based on geometry in your database.

So let us, per experiment, create a shapefile containing all secondary roads of Okinawa.

First we need to prepare an empty directory where this tool can dump our data. Since it will create multiple files, it is best to put them in their own spot.
Open up a terminal window and go to your favorite directory-making place and create a directory called "okinawa-roads":

	:::bash
	$ mkdir okinawa-roads

Next enter that directory.

The "pgsql2shp" tool needs a few parameters to be able to successfully complete. We will be using the following flags:

* -f, tells the tool which file name to adhere
* -u, the database user to connect with

After these flags we need to input the database we wish to take a chunk out of and the query which will determine the actual data to be dumped.

The above will result in the following command:

	:::bash
	$  pgsql2shp -f secundairy_roads -u postgres gis "select way, ref from planet_osm_line where highway = 'secondary';"

As you can see we construct a query which only gets the road reference and the geometry "way" column from the secondary road types.

After some processing it will have created 4 files, the 3 mandatory ones mentioned above, and a new one called a *projection* file.
This file contains the coordinate system and other projection information in WKT format.

This bundle of 4 files is now our shapefile format which you could easily exchange between GIS aware software packages.

### Importing with shp2pgsql

Let us now import these shapefiles back into PostgreSQL and see what happens.

For this we will ignore out "gis" database, and simply create a new database to keep things separated.
Connect to a PostgreSQL terminal, create the database and make it PostGIS aware:

	:::sql
	CREATE DATABASE gisshape;
	\c gisshape
	CREATE EXTENSION postgis;

Now go back to your terminal window to do some importing.

The import tool works by dumping the SQL statements to *stdin* or to a SQL dump file if preferred.
If you do not wish to work with such a dump file, you have to pipe the output to the *psql* command to be able to load in the data.

From the directory where you saved the shapefile dump, run the "shp2pgsql" tool:

	:::bash
	$ shp2pgsql -S -s 900913 -I secundairy_roads | psql -U postgres gisshape

Let me go over the flags we used:

* -S: is used to keep the geometry *simple*. The tool otherwise will convert all geometry to its *MULTI...* counterpart
* -s: is needed to set the correct SRID
* -I: specifies that we wish the tool to create *GiST* indexes on the geometry columns

Note that the *-S* flag will only work if all of your geometry is actual simple and does not contain true MULTI... types of geometry with multiple linestrings, points or polygons in them.

An annoying fact is that you *have* to tell the loader which SRID your geometry is in. There is a *.prj* file in our shapefile bundle, but it only contains the WKT projection information, not the SRID.
One trick to find the SRID based on the information in the projection file is by using *OpenGEO*'s [Prj2EPSG"](http://prj2epsg.org) website, which does quite a good job at looking up the EPSG ID (which most of the time is the SRID). However, it fails to find the SRID of our OSM projection.

Another way of finding our about the SRID is by using the PostGIS *spatial_ref_sys* table itself:

	:::sql
	SELECT srid FROM spatial_ref_sys WHERE srtext = 'PROJCS["Popular Visualisation CRS / Mercator (deprecated)",GEOGCS["Popular Visualisation CRS",DATUM["Popular_Visualisation_Datum",SPHEROID["Popular Visualisation Sphere",6378137,0,AUTHORITY["EPSG","7059"]],TOWGS84[0,0,0,0,0,0,0],AUTHORITY["EPSG","6055"]],PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],UNIT["degree",0.01745329251994328,AUTHORITY["EPSG","9122"]],AUTHORITY["EPSG","4055"]],UNIT["metre",1,AUTHORITY["EPSG","9001"]],PROJECTION["Mercator_1SP"],PARAMETER["central_meridian",0],PARAMETER["scale_factor",1],PARAMETER["false_easting",0],PARAMETER["false_northing",0],AUTHORITY["EPSG","3785"],AXIS["X",EAST],AXIS["Y",NORTH]]';

This will gives us:

	:::bash
	900913
	
Perfect!

If you now connect to your database and query its structure:

	:::sql
	\c gisshape
	\d
	
You will see we have a new table called "secondary_roads". This table now holds only the information we dumped into the shapefile, being our road route numbers and their geometry. Neat!

## The end

Good.

We are done folks. I hope I have given you enough firepower to be able to commence with your own GIS work, using PostGIS.
As I have said in the beginning of this series, the past three chapters form merely an introduction into the capabilities of PostGIS, so as I expect you will do every time: go out and explore!

Try to load in different areas of the world, either with OpenStreetMap or by using shapefiles. Experiment with all the different GIS functions and operators that PostGIS makes available.

And above all, have fun!

And as always...thanks for reading!

<!--  LocalWords:  PostGIS PostgreSQL GIS OpenStreetMap
 -->
