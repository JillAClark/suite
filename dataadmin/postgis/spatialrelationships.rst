.. _dataadmin.postgis.spatialrelationships:

Spatial Relationships
=====================

Functions like ``ST_Area``, ``ST_Length`` or , ``ST_GeomFromText`` only work on one geometry at a time.  But spatial databases are powerful because they not only store geometry, they also have the ability to compare *relationships between geometries*. 

Questions like "Which are the closet bike racks to a park?" or "Where are the intersections of subway lines and streets?" can only be answered by comparing geometries representing the bike racks, streets, and subway lines.

The following set of methods can compare geometries.

ST_Equals
---------
 
``ST_Equals(geometry A, geometry B)`` tests the spatial equality of two geometries. 

.. figure:: img/st_equals.png
   :align: center

   *ST_Equals*

ST_Equals returns TRUE if two geometries of the same type have identical x,y coordinate values, i.e. if the secondary shape is equal (identical) to the primary shape object.

First, let's retrieve a representation of a point.

.. code-block:: sql

  SELECT name, the_geom, ST_AsText(the_geom)
  FROM nyc_subway_stations 
  WHERE name = 'Broad St';             

::

     name   |                      the_geom                      |      st_astext
  ----------+----------------------------------------------------+-----------------------
   Broad St | 0101000020266900000EEBD4CF27CF2141BC17D69516315141 | POINT(583571 4506714)
 
Then, plug the geometry representation back into an ``ST_Equals`` test:

.. code-block:: sql

  SELECT name 
  FROM nyc_subway_stations 
  WHERE ST_Equals(the_geom, '0101000020266900000EEBD4CF27CF2141BC17D69516315141');

::

   Broad St

.. note::

  The representation of the point was not very human readable (``0101000020266900000EEBD4CF27CF2141BC17D69516315141``) but it was an exact representation of the coordinate values. For a test like equality, using the exact coordinates in necessary.  For more on this, please see the section on :ref:`dataadmin.postgis.equality`.


Intersections
-------------

``ST_Intersects``, ``ST_Crosses``, and ``ST_Overlaps`` test whether the interiors of the geometries intersect. 

.. figure:: img/st_intersects.png
   :align: center

   *ST_Intersects*

``ST_Intersects(geometry A, geometry B)`` returns t (TRUE) if the intersection does not result in an empty set.

.. figure:: img/st_disjoint.png
   :align: center

   *ST_Disjoint*

The opposite of ST_Intersects is ``ST_Disjoint(geometry A , geometry B)``. If two geometries are disjoint, they do not intersect, and vice-versa.

.. note:: It is often more efficient to test "not intersects" than to test "disjoint" because the intersects tests can be spatially indexed, while the disjoint test cannot.

.. figure:: img/st_crosses.png  
   :align: center

   *ST_Crosses*

For multipoint/polygon, multipoint/linestring, linestring/linestring, linestring/polygon, and linestring/multipolygon comparisons, ``ST_Crosses(geometry A, geometry B)`` returns t (TRUE) if the intersection results in a geometry whose dimension is one less than the maximum dimension of the two source geometries and the intersection set is interior to both source geometries.

.. figure:: img/st_overlaps.png
   :align: center

   *ST_Overlaps*

``ST_Overlaps(geometry A, geometry B)`` compares two geometries of the same dimension and returns TRUE if their intersection set results in a geometry different from both but of the same dimension.

For example, given a dataset containing New York City subways and neighborhoods, it is possible to determine a subway station's neighborhood using the ``ST_Intersects`` function:

.. code-block:: sql

  SELECT name, ST_AsText(the_geom)
  FROM nyc_subway_stations 
  WHERE name = 'Broad St';               

::

  POINT(583571 4506714)

.. code-block:: sql   

  SELECT name, boroname 
  FROM nyc_neighborhoods
  WHERE ST_Intersects(the_geom, ST_GeomFromText('POINT(583571 4506714)',26918));

::

          name        | boroname  
  --------------------+-----------
   Financial District | Manhattan



Touching
--------

``ST_Touches`` tests whether two geometries touch at their boundaries, but do not intersect in their interiors 

.. figure:: img/st_touches.png
   :align: center

   *ST_Touches*

``ST_Touches(geometry A, geometry B)`` returns TRUE if either of the geometries' boundaries intersect or if only one of the geometry's interiors intersects the other's boundary.

Containing
----------

``ST_Within`` and ``ST_Contains`` test whether one geometry is fully within the other. 

.. figure:: img/st_within.png
   :align: center

   *ST_Within*
    
``ST_Within(geometry A, geometry B)`` returns TRUE if the first geometry is completely within the second geometry. ST_Within tests for the exact opposite result of ST_Contains.  

``ST_Contains(geometry A, geometry B)`` returns TRUE if the second geometry is completely contained by the first geometry. 


Distance
--------

An extremely common GIS question is "find all the stuff within distance X of this other stuff". 

The ``ST_Distance(geometry A, geometry B)`` calculates the (shortest) distance between two geometries and returns it as a float. This is useful for actually reporting back the distance between objects.

.. code-block:: sql

  SELECT ST_Distance(
    ST_GeometryFromText('POINT(0 5)'),
    ST_GeometryFromText('LINESTRING(-2 2, 2 2)'));

::

  3

For testing whether two objects are within a distance of one another, the ``ST_DWithin`` function provides an index-accelerated true/false test. This is useful for questions like "how many trees are within a 500 meter buffer of the road?". You don't have to calculate an actual buffer, you just have to test the distance relationship.

  .. figure:: img/st_dwithin.png
     :align: center

     *ST_DWithin*
    
Given the hypothetcial NYC data, one can find the streets nearby (within 10 meters of) the subway stop:

.. code-block:: sql

  SELECT name 
  FROM nyc_streets 
  WHERE ST_DWithin(
          the_geom, 
          ST_GeomFromText('POINT(583571 4506714)',26918), 
          10
        );

:: 

       name     
  --------------
     Wall St
     Broad St
     Nassau St

And we can verify the answer on a map. The Broad St station is actually at the intersection of Wall, Broad and Nassau Streets.


For more information about geometry functions in PostGIS, please see the `PostGIS Reference <../../../postgis/postgis/html/reference.html>`_