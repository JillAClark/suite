.. _dataadmin.loading.shp2pgsql:


Loading data into PostGIS from the terminal
===========================================

PostGIS comes with a utility called ``shp2pgsql`` for converting shapefiles into database tables.  This section will show how to use this utility to load a single file or multiple files.


How It Works
------------

The command ``shp2pgsql`` converts a shapefile into SQL commands that can then be loaded into a database.  It does not perform the actual loading.  The output of this command can be captured into a SQL file, or piped directly into the ``psql`` command which will run the commands against a database.

Preparation
-----------

#. First pick a shapefile that you wish to load.  You need all the required files: ``.shp``, ``.shx``, and ``.dbf``.

#. Determine the SRID ("projection") of your data.  This is often determined in the layer metadata.  If not known, use a service like `prj2epsg.org <http://prj2epsg.org>`_ to upload and convert the ``.prj`` file in the shapefile to an SRID code.

#. Determine into which database you would like to load the data, or create a new database.  The OpenGeo Suite comes with a default database that can be used, which is named for your username on the system (or the user that installed the OpenGeo Suite).

Loading data
------------

#. Open up a terminal.  If the utilities ``shp2pgsql`` and ``psql`` aren't on your path, you may wish to add them now for easier use.

#. Check that PostGIS is responding properly.  In this context, the easiest way to test this is to run a ``psql`` query:

   .. code-block:: console

      psql -p 54321 -c "SELECT PostGIS_Version()"

   .. code-block:: console

                  postgis_version
      ---------------------------------------
       2.0 USE_GEOS=1 USE_PROJ=1 USE_STATS=1

   .. note::

     These examples will use port 54321, but substitute your PostGIS port if different.  Also, if your connection is denied, you may need to add your user name with the ``-U`` option or set the hostname with the ``-h`` option.

#. Run ``shp2pgsql`` command, and pipe it into the ``psql`` command to load the shapefile in the database in one step.  There are many options in the ``shp2pgsql`` command; here is a recommended syntax:

   .. code-block:: console

      shp2pgsql -I -s <SRID> <PATH/TO/SHAPEFILE> <DBTABLE> | psql -d <DATABASE>

   where:

   * ``<SRID>`` is the SRID defined above.
   * ``<PATH/TO/SHAPEFILE.SHP>`` is the full path to the shapefile (such as :file:`C:\\MyData\\roads\\roads.shp`)
   * ``<DBTABLE>`` is the name of the newly created database table, usually the same as the shapefile filename
   * ``<DATABASE>`` is the name of the database where the table will be created

   For example:

   .. code-block:: console

      shp2pgsql -I -s 4269 C:\MyData\roads\roads.shp roads | psql -p 54321 -d MyDatabase

   The ``-I`` option will create a spatial index after the table is created.  This is strongly recommended for performance purposes.

#. If you want to capture the SQL commands into a file, you can do this by piping the output to a file:

   .. code-block:: console

      shp2pgsql -I -s <SRID> <PATH/TO/SHAPEFILE> <DBTABLE> > SHAPEFILE.sql

   The file can later be loaded into the database by running:

   .. code-block:: console

      psql -p 54321 -d <DATABASE> -f SHAPEFILE.sql

The shapefile has now been imported as a table in your PostGIS database.  You can verify this in pgAdmin by navigating to your database and viewing the list of tables.  You can also verify the table creation on the command line by typing:

.. code-block:: console

   psql -p <PORT> -U <USERNAME> -d <DATABASE> -c "\d"

Replace the variables in the above command with the correct values for your system.

.. code-block:: console

      Schema |         Name         |   Type   |  Owner
     --------+----------------------+----------+----------
      public | bc_2m_border         | table    | postgres
      public | bc_2m_border_gid_seq | sequence | postgres
      public | geometry_columns     | table    | postgres
      public | spatial_ref_sys      | table    | postgres

If you need to load more shapefiles, you may repeat this process.

Batch loading
-------------

Like all command line utilities, the ``shp2pgsql`` command can be wrapped in batch operations.  The following will show how to do this in a few contexts:

Windows Command (Batch)
~~~~~~~~~~~~~~~~~~~~~~~

.. note:: This script assumes that all the files have the same projection.

Create a batch file (:file:`loadfiles.cmd`) in the same directory as the shapefiles to be loaded.  Add the following content:

.. code-block:: console

   for %%f in (*.shp) do shp2pgsql -I -s <SRID> %%f %%~nf > %%~nf.sql
   for %%f in (*.sql) do psql -p <PORT> -d <DATABASE> -f %%f

Run this command to load all shapefiles into the database.

Bash
~~~~

.. note:: This script assumes that all the files have the same projection.

Create a shell script file (:file:`loadfiles.sh`) in the same directory as the shapefiles to be loaded.  Add the following content:

.. code-block:: console

   #!/bin/bash

   for f in *.shp
   do
       shp2pgsql shp2pgsql -I -s <SRID> $f `basename $f .shp` > `basename $f .shp`.sql
   done

   for f in *.sql
   do
       psql -p <PORT> -d <DATABASE> -f $f
   done



