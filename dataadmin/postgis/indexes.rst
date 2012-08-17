.. _dataadmin.postgis.indexes:

Spatial Indexes
===============

The "spatial index" is one of the three key features of a spatial database. Without indexing, any search for a feature would require a "sequential scan" of every record in the database. Indexing speeds up searching by organizing the data into a search tree which can be quickly traversed to find a particular record. 

Spatial indexes are one of the greatest assets of PostGIS.  Comparing whole tables with each other can get very costly: joining two tables of 10,000 records each without indexes would require 100,000,000 comparisons!  With indexes the cost could be as low as 20,000 comparisons.

When loading data, a spatial index is typically created called ``<TABLENAME>_the_geom_gist``.  If this is not created, you can create one manually:

.. code-block:: sql

  CREATE INDEX <TABLENAME>_the_geom_gist ON <TABLENAME> USING GIST (the_geom);

.. note:: The ``USING GIST`` clause tells PostgreSQL to use the generic index structure (GiST) when building the index.

Running a complex query when there is an index on a table can potentially save tremendous amounts of processing time.


How Spatial Indexes Work
------------------------

Standard database indexes create a hierarchical tree based on the values of the column being indexed. Spatial indexes are a little different -- they are unable to index the geometric features themselves and instead index the bounding boxes of the features.

.. figure:: img/indexes_bbox.png
   :align: center

   *Comparing bounding boxes*

In the figure above, the number of lines that intersect the yellow star is **one**, the red line. But the bounding boxes of features that intersect the yellow box is **two**, the red and blue ones.

The way the database efficiently answers the question "what lines intersect the yellow star?" is by a "two-pass" system.

#. Answer the question "what boxes intersect the yellow box?" using the index (which is very fast)
#. Do an exact calculation of "what lines intersect the yellow star" but **only for those features returned by the first test** 

For a large table, this "two pass" system of evaluating the approximate index first, then carrying out an exact test can radically reduce the amount of calculations necessary to answer a query.

Both PostGIS and Oracle Spatial share the same "R-Tree" spatial index structure. R-Trees break up data into rectangles, and sub-rectangles, and sub-sub rectangles, etc.  It is a self-tuning index structure that automatically handles variable data density and object size.

.. figure:: img/indexes_rtree.png
   :align: center

   *R-tree Hierarchy for geometries*

Index-Only Queries
------------------

Most of the commonly used functions in PostGIS (``ST_Contains``, ``ST_Intersects``, ``ST_DWithin``, etc.) include an index filter automatically. But some functions (such as ``ST_Relate``) do not include an index filter.

To do a bounding-box search using the index (and no filtering), make use of the ``&&`` operator. For geometries, the ``&&`` operator means "bounding boxes overlap or touch" in the same way that for number the ``=`` operator means "values are the same".

Let's compare an index-only query for the population of the 'West Village' to a more exact query. Using ``&&`` our index-only query looks like the following:

.. code-block:: sql

  SELECT Sum(popn_total) 
  FROM nyc_neighborhoods neighborhoods
  JOIN nyc_census_blocks blocks
  ON neighborhoods.the_geom && blocks.the_geom
  WHERE neighborhoods.name = 'West Village';
  
::

  50325
  
Now let's do the same query using the more exact ``ST_Intersects`` function.

.. code-block:: sql

  SELECT Sum(popn_total) 
  FROM nyc_neighborhoods neighborhoods
  JOIN nyc_census_blocks blocks
  ON ST_Intersects(neighborhoods.the_geom, blocks.the_geom)
  WHERE neighborhoods.name = 'West Village';
  
::

  27141

A much lower answer! The first query summed up every block that intersected the neighborhood's bounding box; the second query only summed up those blocks that intersected the neighborhood itself.


Analyzing
---------

Counter-intuitively, it is not always faster to do an index search: if the search is going to return every record in the table, traversing the index tree to get each record will actually be slower than just linearly reading the whole table from the start.

The PostgreSQL query planner intelligently chooses when to use or not to use indexes to evaluate a query.  In order to figure out what situation it is dealing with (reading a small part of the table versus reading a large portion of the table), PostgreSQL keeps statistics about the distribution of data in each indexed table column.  By default, PostgreSQL gathers statistics on a regular basis. However, if you dramatically change the make-up of your table within a short period of time, the statistics will not be up-to-date.

To ensure your statistics match your table contents, it is wise the to run the ``ANALYZE`` command after bulk data loads and deletes in your tables. This force the statistics system to gather data for all your indexed columns.

The ``ANALYZE`` command asks PostgreSQL to traverse the table and update its internal statistics used for query plan estimation (query plan analysis will be discussed later). 

.. code-block:: sql

   ANALYZE nyc_census_blocks;
   
Vacuuming
---------

Just creating an index is not enough to allow PostgreSQL to use it effectively.  VACUUMing must be performed whenever a new index is created or after a large number of UPDATEs, INSERTs or DELETEs are issued against a table.  The ``VACUUM`` command asks PostgreSQL to reclaim any unused space in the table pages left by updates or deletes to records. 

Vacuuming is so critical for the efficient running of the database that PostgreSQL provides an "autovacuum" option.

Enabled by default, autovacuum both VACUUMs (recovers space) and ANALYZEs (updates statistics) on your tables at sensible intervals determined by the level of activity.  While this is essential for highly transactional databases, it is not advisable to wait for an autovacuum run after adding indexes or bulk-loading data.  If a large batch update is performed, you should manually run ``VACUUM``.

Vacuuming and analyzing the database can be performed separately as needed.  Issuing ``VACUUM`` command will not update the database statistics; likewise issuing an ``ANALYZE`` command will not recover unused table rows.  Both commands can be run against the entire database, a single table, or a single column.

.. code-block:: sql

   VACUUM ANALYZE nyc_census_blocks;

