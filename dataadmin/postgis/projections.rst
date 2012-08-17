.. _dataadmin.postgis.projection:

Projections
===========

The earth is not flat, and there is no simple way of putting it down on a flat paper map (or computer screen), so people have come up with all sorts of ingenious solutions, each with pros and cons. Some projections preserve area, so all objects have a relative size to each other; other projections preserve angles (conformal) like the Mercator projection; some projections try to find a good intermediate mix with only little distortion on several parameters. Common to all projections is that they transform the (spherical) world onto a flat Cartesian coordinate system, and which projection to choose depends on how you will be using the data.

Sometimes, however, you need to transform and re-project between spatial reference systems. PostGIS includes built-in support for changing the projection of data, using the ``ST_Transform(geometry, srid)`` function. For managing the spatial reference identifiers on geometries, PostGIS provides the ``ST_SRID(geometry)`` and ``ST_SetSRID(geometry, srid)`` functions.

We can confirm the SRID of data with the ``ST_SRID`` command:

.. code-block:: sql

   SELECT ST_SRID(the_geom) FROM some_table LIMIT 1;
  
::

  26918
  
The definition of "26918" (or indeed of any valid SRID value) is contained in the ``spatial_ref_sys`` table. In fact, **two** definitions are there. The "well-known text" (``WKT``) definition is in the ``srtext`` column, and there is a second definition in "proj.4" format in the ``proj4text`` column.

.. code-block:: sql

   SELECT * FROM spatial_ref_sys WHERE srid = 26918;
   
In fact, for the internal PostGIS re-projection calculations, it is the contents of the ``proj4text`` column that are used. For our 26918 projection, here is the proj.4 text:

.. code-block:: sql

  SELECT proj4text FROM spatial_ref_sys WHERE srid = 26918;
  
::

  +proj=utm +zone=18 +ellps=GRS80 +datum=NAD83 +units=m +no_defs 
  
In practice, both the ``srtext`` and the ``proj4text`` columns are important: the ``srtext`` column is used by external programs like `GeoServer <../../../geoserver>`_, `uDig <http://udig.refractions.net>`_, `FME <http://www.safe.com/>`_  and others; the ``proj4text`` column is used internally.

Comparing Data
--------------

Taken together, a coordinate and an SRID define a location on the globe. Without an SRID, a coordinate is just an abstract notion. A "Cartesian" coordinate plane is defined as a "flat" coordinate system placed on the surface of Earth. Because PostGIS functions work on such a plane, comparison operations require that both geometries be represented in the same SRID.

If you feed in geometries with differing SRIDs you will just get an error:

.. code-block:: sql

  SELECT ST_Equals(
           ST_GeomFromText('POINT(0 0)', 4326),
           ST_GeomFromText('POINT(0 0)', 26918)
           );

::

  ERROR:  Operation on two geometries with different SRIDs
  CONTEXT:  SQL function "st_equals" statement 1
  

There is a function for transforming data, ``ST_Transform``.  But be careful of using it for on-the-fly conversion, as spatial indexes are built using SRID of the stored geometries.  If a comparison is done in a different SRID, spatial indexes are often not used. **It is best practice to choose one SRID for all the tables in your database.** Only use the transformation function when you are reading or writing data to external applications.


Transforming Data
-----------------

If we return to our proj4 definition for SRID 26918, we can see that our working projection is UTM (Universal Transverse Mercator) of zone 18, with meters as the unit of measurement.

::

   +proj=utm +zone=18 +ellps=GRS80 +datum=NAD83 +units=m +no_defs 

Let's convert some data from our working projection to geographic coordinates -- also known as "longitude/latitude". 

To convert data from one SRID to another, you must first verify that your geometry has a valid SRID. Since we have already confirmed a valid SRID, we next need the SRID of the projection to transform into. In other words, what is the SRID of geographic coordinates?

The most common SRID for geographic coordinates is 4326, which corresponds to "longitude/latitude on the WGS84 spheroid". You can see the definition at `spatialreference.org <http://spatialreference.org/ref/epsg/4326/>`_.

You can also pull the definitions from the ``spatial_ref_sys`` table:

.. code-block:: sql

  SELECT srtext FROM spatial_ref_sys WHERE srid = 4326;
  
::

  GEOGCS["WGS 84",
    DATUM["WGS_1984",
      SPHEROID["WGS 84",6378137,298.257223563,AUTHORITY["EPSG","7030"]],
      AUTHORITY["EPSG","6326"]],
    PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],
    UNIT["degree",0.01745329251994328,AUTHORITY["EPSG","9122"]],
    AUTHORITY["EPSG","4326"]]

If we would convert the coordinates of the 'Broad St' subway station into geographics, we would get this:

.. code-block:: sql

  SELECT ST_AsText(ST_Transform(the_geom,4326)) 
  FROM nyc_subway_stations 
  WHERE name = 'Broad St';
  
::

  POINT(-74.0106714688735 40.7071048155841)

To view a table's SRID assignment, query the database's ``geometry_columns`` table.

.. code-block:: sql

  SELECT f_table_name AS name, srid 
  FROM geometry_columns;
  
::

          name         | srid  
  ---------------------+-------
   nyc_census_blocks   | 26918
   nyc_neighborhoods   | 26918
   nyc_streets         | 26918
   nyc_subway_stations | 26918
   geometries          |    -1

If you load data or create a new geometry without specifying an SRID, the SRID value will be -1.

  
Updating the SRID
-----------------

Sometimes, when loading data into PostGIS, the data was loaded properly, but the SRID wasn't set.  In order to change this, run the following command:

.. code-block:: sql

  SELECT UpdateGeometrySRID('some_table', 'the_geom', <srid>);

This will update the SRID in the ``geometry_columns``, to the value of ``<srid>``, but leave the data itself unchanged.