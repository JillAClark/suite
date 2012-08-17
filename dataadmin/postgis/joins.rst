.. _dataadmin.postgis.joins:

Spatial Joins
=============

Spatial joins are the bread-and-butter of spatial databases.  They allow you to combine information from different tables by using :ref:`dataadmin.postgis.spatialrelationships` as the join key.  Much of what we think of as "standard GIS analysis" can be expressed as spatial joins.

The example in the section on :ref:`dataadmin.postgis.spatialrelationships` used a two-step process to find out information.  Using a spatial join, it is possible to answer the question in a single query.  

Given a subway station location, neighborhood data, and borough names, we can find the neighborhood that contains the subway station:

.. code-block:: sql

  SELECT 
    subways.name AS subway_name, 
    neighborhoods.name AS neighborhood_name, 
  FROM nyc_neighborhoods AS neighborhoods
  JOIN nyc_subway_stations AS subways
  ON ST_Contains(neighborhoods.the_geom, subways.the_geom)
  WHERE subways.name = 'Broad St';

:: 

   subway_name | neighborhood_name  
  -------------+--------------------
   Broad St    | Financial District 

Any function that provides a true/false relationship between two tables can be used to drive a spatial join, but the most commonly used ones are: ``ST_Intersects``, ``ST_Contains``, and ``ST_DWithin``.

Join and Summarize
------------------

The combination of a ``JOIN`` with a ``GROUP BY`` provides the kind of analysis that is usually done in a GIS system.

For example, consider the question: 

  "What is the population and racial make-up of the neighborhoods of Manhattan?"

Here we have a question that combines information from about population from the census with the boundaries of neighborhoods, with a restriction to just one borough of Manhattan.

.. code-block:: sql

  SELECT 
    neighborhoods.name AS neighborhood_name, 
    Sum(census.popn_total) AS population,
    Round(100.0 * Sum(census.popn_white) / Sum(census.popn_total),1) AS white_pct,
    Round(100.0 * Sum(census.popn_black) / Sum(census.popn_total),1) AS black_pct
  FROM nyc_neighborhoods AS neighborhoods
  JOIN nyc_census_blocks AS census
  ON ST_Intersects(neighborhoods.the_geom, census.the_geom)
  WHERE neighborhoods.boroname = 'Manhattan'
  GROUP BY neighborhoods.name
  ORDER BY white_pct DESC;

::

   neighborhood_name  | population | white_pct | black_pct 
 ---------------------+------------+-----------+-----------
  Carnegie Hill       |      19909 |      91.6 |       1.5
  North Sutton Area   |      21413 |      90.3 |       1.2
  West Village        |      27141 |      88.1 |       2.7
  Upper East Side     |     201301 |      87.8 |       2.5
  Greenwich Village   |      57047 |      84.1 |       3.3
  Soho                |      15371 |      84.1 |       3.3
  Murray Hill         |      27669 |      79.2 |       2.3
  Gramercy            |      97264 |      77.8 |       5.6
  Central Park        |      49284 |      77.8 |      10.4
  Tribeca             |      13601 |      77.2 |       5.5
  Midtown             |      70412 |      75.9 |       5.1
  Chelsea             |      51773 |      74.7 |       7.4
  Battery Park        |       9928 |      74.1 |       4.9
  Upper West Side     |     212499 |      73.3 |      10.4
  Financial District  |      17279 |      71.3 |       5.3
  Clinton             |      26347 |      64.6 |      10.3
  East Village        |      77448 |      61.4 |       9.7
  Garment District    |       6900 |      51.1 |       8.6
  Morningside Heights |      41499 |      50.2 |      24.8
  Little Italy        |      14178 |      39.4 |       1.2
  Yorkville           |      57800 |      31.2 |      33.3
  Inwood              |      50922 |      29.3 |      14.9
  Lower East Side     |     104690 |      28.3 |       9.0
  Washington Heights  |     187198 |      26.9 |      16.3
  East Harlem         |      62279 |      20.2 |      46.2
  Hamilton Heights    |      71133 |      14.6 |      41.1
  Chinatown           |      18195 |      10.3 |       4.2
  Harlem              |     125501 |       5.7 |      80.5


In this example:

#. The ``JOIN`` clause creates a virtual table that includes columns from both the neighborhoods and census tables. 
#. The ``WHERE`` clause filters our virtual table to just rows in Manhattan. 
#. The remaining rows are grouped by the neighborhood name and fed through the aggregation function to ``Sum()`` the population values.
#. After a little arithmetic and formatting (e.g., ``GROUP BY``, ``ORDER BY``) on the final numbers, our query spits out the percentages.

.. note:: 

   The ``JOIN`` clause combines two ``FROM`` items.  By default, we are using an ``INNER JOIN``, but there are four other types of joins. For further information see the `join_type <http://www.postgresql.org/docs/9.1/interactive/sql-select.html>`_ definition in the PostgreSQL documentation.

We can also use distance tests as a join key, to create summarized "all items within a given radius" queries. Let's explore the racial geography of New York using distance queries.

First, let's get the baseline racial make-up of the city.

.. code-block:: sql

  SELECT 
    100.0 * Sum(popn_white) / Sum(popn_total) AS white_pct, 
    100.0 * Sum(popn_black) / Sum(popn_total) AS black_pct, 
    Sum(popn_total) AS popn_total
  FROM nyc_census_blocks;

:: 

        white_pct      |      black_pct      | popn_total 
  ---------------------+---------------------+------------
   44.6586020115685295 | 26.5945063345703034 |    8008278


So, of the 8M people in New York, about 44% are "white" and 26% are "black". 

Duke Ellington once sang that "You / must take the A-train / To / go to Sugar Hill way up in Harlem." As we saw above, Harlem has far and away the highest African-American population in Manhattan (80.5%). Is the same true of Duke's A-train?

First, note that the contents of the ``nyc_subway_stations`` table ``routes`` field is what we are interested in to find the A-train. The values in there are a little complex.

Given a nyc_subway_stations table"

.. code-block:: sql

  SELECT DISTINCT routes FROM nyc_subway_stations;
  
:: 

 A,C,G
 4,5
 D,F,N,Q
 5
 E,F
 E,J,Z
 R,W
 ...

.. note::

   The ``DISTINCT`` keyword eliminates duplicate rows from the result.  Without the ``DISTINCT`` keyword, the query above identifies 491 results instead of 73.
   
So to find the A train, we will want any row in ``routes`` that has an 'A' in it. While there is more than one way to do this, we can use the fact that ``strpos(routes,'A')`` will return a non-zero number if 'A' is in the routes field.

.. code-block:: sql

   SELECT DISTINCT routes 
   FROM nyc_subway_stations AS subways 
   WHERE strpos(subways.routes,'A') > 0;
   
::

  A,B,C
  A,C
  A
  A,C,G
  A,C,E,L
  A,S
  A,C,F
  A,B,C,D
  A,C,E
  
So to summarize the racial make-up of within 200 meters of the A-train line.

.. code-block:: sql

  SELECT 
    100.0 * Sum(popn_white) / Sum(popn_total) AS white_pct, 
    100.0 * Sum(popn_black) / Sum(popn_total) AS black_pct, 
    Sum(popn_total) AS popn_total
  FROM nyc_census_blocks AS census
  JOIN nyc_subway_stations AS subways
  ON ST_DWithin(census.the_geom, subways.the_geom, 200)
  WHERE strpos(subways.routes,'A') > 0;

::

        white_pct      |      black_pct      | popn_total 
  ---------------------+---------------------+------------
   42.0805466940877366 | 23.0936148851067964 |     185259

So the racial make-up along the A-train isn't radically different from the make-up of New York City as a whole. 

