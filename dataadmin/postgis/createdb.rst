.. _dataadmin.postgis.createdb:

Creating a Spatial Database
===========================

PostgreSQL (the database system underlying PostGIS) has the notion of a **template database** that can be used to initialize a new database.  The new database automatically will get a copy of everything from the template. When you installed PostGIS, a spatially enabled database called ``template_postgis`` was created. If we use ``template_postgis`` as a template when creating our new database, the new database will be spatially enabled.

#. Open the **Databases** folder in the pgAdmin tree and have a look at the available databases.  The ``postgres`` database is the user database for the default postgres user and is not too interesting to us.  The ``template_postgis`` database is what you can use to create spatial databases.

#. Right-click on the ``Databases`` item and select ``New Database``.

   .. figure:: img/createdb_newdb.png
      :align: center

      *Creating a new database in pgAdmin*

   .. note:: If you receive an error indicating that the source database (``template_postgis``) is being accessed by other users, this is likely because you still have it selected.  Select another database, then right-click on the ``PostGIS`` item and select :guilabel:`Disconnect`.  Double-click the same item to reconnect and try again.

#. When creating the new database, make sure to select ``template_postgis`` as the :guilabel:`Template`.

   .. figure:: img/createdb_newdbtemplate.png
      :align: center

      *Creating a new database in pgAdmin from template_postgis*

#. Select the new database and open it up to display the tree of objects. Inside the ``public`` schema, you'll see two PostGIS-specific metadata tables: ``geometry_columns`` and ``spatial_ref_sys`` (see the section on :ref:`dataadmin.postgis.metatables`).

   .. figure:: img/createdb_metatables.png
      :align: center

      *Spatial metadata tables*

  .. note:: If you don't see these tables, your database was not created correctly.

#. Click on the SQL query button (or go to :menuselection:`Tools --> Query Tool`).

   .. figure:: img/createdb_querybutton.png
      :align: center

      *Query Tool*

#. Enter the following query into the query text field.  Click the :guilabel:`Play` button (or press **F5**) to "Execute the query."

   .. code-block:: sql

      SELECT postgis_full_version();

   `postgis_full_version() <../../../postgis/postgis/html/PostGIS_Full_Version.html>`_ is a management function that returns version and build configuration.  If this command executes successfully, the database is spatially enabled!


Creating a spatial database on the command line
-----------------------------------------------

You can also create a database from the command line via the ``createdb`` command::

  createdb -p <PORT> -T template_postgis <DATABASENAME>


