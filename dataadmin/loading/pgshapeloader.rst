.. _dataadmin.loading.pgshapeloader:


Loading data into PostGIS with pgShapeloader
============================================

The pgShapeloader utility (also known as "PostGIS Shapefile Import/Export manager" or "PostGIS Shapefile and DBF loader") is a graphical interface for converting shapefiles into PostGIS database tables.  It is a graphical equivalent/companion to the command line ``shp2pgsql`` utility.

.. note::  On Linux systems, pgShapeloader is known as ``shp2pgsql-gui``.


How it works
------------

Like all graphical utilities, pgShapeloader is designed for ease of use, while at the same time retaining broad functionality of the more flexible command line tool.  The basic premise of data loading with pgShapeloader is the same as with the command line tool, :ref:`shp2pgsql <dataadmin.loading.shp2pgsql>`.  The process consists of two steps:

#. Convert shapefile data to a series of SQL commands.
#. Run those commands against the database.

These are accomplished in one step through this utility.

Preparation
-----------

Here are some initial steps:

#. First pick a shapefile that you wish to load.  You need all the required files: ``.shp``, ``.shx``, and ``.dbf``.

#. Determine the SRID ("projection") of your data.  This is often determined in the layer metadata.  If not known, use a service like `prj2epsg.org <http://prj2epsg.org>`_ to upload and convert the ``.prj`` file in the shapefile to an SRID code.

#. Determine into which database you would like to load the data, or create a new database.  The OpenGeo Suite comes with a default database that can be used, which is named for your username on the system (or the user that installed the OpenGeo Suite).


Loading data
------------

#. Make sure the OpenGeo Suite is running.

#. Run pgShapeloader.  This utility can be loaded in a few different ways:

   #. Through the :ref:`dashboard` (though not the web-based Dashboard)
   #. Through pgAdmin by going to the :guilabel:`Plugins` menu and selecting :guilabel:`PostGIS Shapefile and DBF loader`.
   #. Through the command line by running :file:`shp2pgsql-gui`.

   .. figure:: img/pgshapeloader.png

      *pgShapeloader*

#. Click on the :guilabel:`View connection details` button and fill out the :guilabel:`PostGIS Connection` section.  This information will depend on how the OpenGeo Suite was installed, and whether pgShapeloader is running locally or remotely.  Given a default local installation, the following credentials can be used:

   .. list-table::
      :header-rows: 1

      * - Field
        - Windows/Mac
        - Linux
      * - **Username**
        - postgres
        - opengeo
      * - **Password**
        - [blank/any]
        - opengeo
      * - **Server Host**
        - localhost
        - localhost
      * - **Port**
        - **54321**
        - **5432**
      * - **Database**
        - [user name]
        - [user name]

   .. figure:: img/pgshp_connection.png

      *PostGIS connection options*

#. Click :guilabel:`OK` to return to the main application.  The connection parameters will be tested, and the results will be shown in the :guilabel:`Log Window`.  If you see errors, check your parameters and try again.

   .. figure:: img/pgshp_connectionsuccess.png

      *A successful PostGIS connection*

#. Now select your source files by clicking in the box titled :guilabel:`Add File`.  In the folder dialog that appears, navigate to the location of your shapefile, select it, then click :guilabel:`Open`.  Multiple files can also be added in this way

   .. figure:: img/pgshp_select.png

      *Selecting a shapefile to import*

#. Edit the configuration information in :guilabel:`Import List`.  Most information can be kept as the the default, but it is important to **enter the correct SRID** for each layer.

   .. warning:: Make sure not to skip this step, otherwise your data won't be loaded properly.

   .. figure:: img/pgshp_srid.png

      *Changing the SRID*

#. Clicking on the :guilabel:`Options` button will bring up some additional configuration items.  If you are not sure, you can safely leave these as the defaults.

   .. list-table::
      :header-rows: 1

      * - Option
        - Description
      * - DBF file character encoding
        - Specifies the character encoding of the shapefile's attribute columns.  Default is **UTF-8**.
      * - Preserve case of column names
        - When unchecked, the case of column names will be made all lowercase.  When checked, mixed case will be preserved.
      * - Do not create 'bigint' columns
        - Columns with type 'bigint' will not be created.
      * - Create spatial index automatically after load
        - Creates a spatial index automatically.  Indexing is highly recommended for performance reasons, but you can uncheck this option if you wish to create the index manually or if you're sure you don't want one.
      * - Load only attribute (dbf) data
        - Strips the geometry column from the loading process, leaving just the attribute columns.  Default is unchecked.
      * - Load data using COPY rather than INSERT
        - This can sometimes make the load process a little faster.
      * - Load into GEOGRAPHY column
        - Will load the geospatial data as type GEOGRAPHY instead of the default of GEOMETRY.  Requires lat/lon data (often known as SRID 4326).
 
   .. figure:: img/pgshp_options.png

      *Additional options*

#. Now that the form is complete, click :guilabel:`Import` to start the conversion.

   .. figure:: img/pgshp_importing.png
      :align: center

      *Import in progress*

#. The shapefile has been imported as a table in your PostGIS database.

   .. figure:: img/pgshp_success.png
      :align: center

      *A successful import*

#.  You can verify this in pgAdmin by navigating to your database and viewing the list of tables.  Your shapefile's name should be listed there.

   .. figure:: img/pgshp_pgadminconfirm.png
      :align: center

      *Confirming import in pgAdmin*

#. You can also verify the table creation on the command line by typing:

   .. code-block:: console

      psql -p <PORT> -U <USERNAME> -d <DATABASE> -c "\d"

   Replace the variables in the above command with the correct values for your system.

   .. code-block:: console

      Schema |         Name         |   Type   |  Owner
     --------+----------------------+----------+----------
      public | bc_2m_border         | table    | postgres
      public | bc_2m_border_gid_seq | sequence | postgres
      public | geography_columns    | view     | postgres
      public | geometry_columns     | table    | postgres
      public | raster_columns       | table    | postgres
      public | raster_overviews     | table    | postgres
      public | spatial_ref_sys      | table    | postgres

