.. _dataadmin.loading.gsrest:


Loading data into GeoServer via REST
====================================

GeoServer has a robust REST-like interface for performing operations without the need to go through the web-based administration pages.  This section will describe some common data loading paths via the REST interface, including uploading a shapefile directly into GeoServer as well as publishing existing PostGIS tables in GeoServer.

.. note:: For more information about the GeoServer REST interface, please see the `GeoServer Documentation <../../geoserver/restconfig/>`_.


Some tips about REST
--------------------

There are four types of requests:

* **GET** requests are used to query the server for information.
* **POST** requests are used to create new information.
* **PUT** requests are used to update existing information.
* **DELETE** is used to delete information.

When changing information on the server, if a POST request is not working, try PUT, or vice versa.

As a rule of thumb, when unsure about how to create the actual request content for use in the POST/PUT request, try GETting a request for a similar endpoint (existing workspace, store, layer) and viewing the resulting output.  That output can be a good model for how to create your input.


Prerequisites
-------------

There are many ways to interact with the REST interface.  `cURL <http://curl.haxx.se/>`_ is a cross-platform command-line utility that allows for basic interaction.  Specialized REST clients also exist, both as plugins for browsers or as standalone utilities.

The examples below will use cURL, though they can be adapted to other clients easily.

#. Install cURL on your operating system, if it is not already installed.

#. Add cURL to your path.  Ensure this is set up correctly by running:

   .. code-block:: console

      curl --version

#. Issue a test command:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest

   .. note:: All of the commands in this section will use ``http://localhost:8080/geoserver/`` for the GeoServer path, ``admin`` for the username, and ``geoserver`` for the password.  You will need to alter these commands to match your configuration.

   You should receive a listing of resources (workspaces, styles, layers, etc.) in a raw HTML file.  If you receive an error, ensure that GeoServer is responding (by navigating to the web admin page), and that you have entered the correct GeoServer path and credentials.


Uploading a shapefile to GeoServer 
----------------------------------

With a single command, it is possible to upload a shapefile to GeoServer.  This command embeds a zipped shapefile in the body of a request and transfers it to GeoServer.  GeoServer then creates the all the necessary catalog entries (shapefile store, featuretype) and publishes the layer.

#. First, create a ZIP file of the shapefile's components (the ``.shp``, ``.shx``, ``.dbf``, and ``.prj`` files).

#. Determine the workspace where the layer should be associated.  For the a list of workspaces, issue this command:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest/workspaces.xml

   For just the default workspace, you can run:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest/workspaces/default.xml

   The names of the workspaces will be inside the ``<name>`` tag.  Make note of which one you want to use.  These steps will use a workspace called ``opengeo``, the default in the OpenGeo Suite.

#. Construct the shapefile upload request, in this case, a file called :file:`border.zip`:

   .. code-block:: console

      curl -v -u admin:geoserver -X PUT -H "Content-type: application/zip" --data-binary @border.zip http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores/borderstore/file.shp



   Some notes on this command:
 
   * The new layer name will be the same name as the zip file (``border``).
   * The new store name will be the string after datastores and before ``file.shp`` (so in the above example, ``borderstore``).
   * The endpoint **must be named** ``file.shp``, and not the name of the shapefile being uploaded.  It is not necessary to rename the shapefile.
   * If you have to enter path information to the shapefile archive, enclose the path in double quotes:  ``@"C:\data\data.zip"``
   * This command uses a PUT request instead of a POST because it is editing an existing resource (the list of datastores).

#. If the command was run successfully, you should see within the output:

   .. code-block:: console

      < HTTP/1.1 201 Created

   If you see a ``500 Internal Server Error`` or ``405 Method Not Allowed``, or any other error, then the command was issued incorrectly, and you should verify the syntax and content.  Examine the output and logs for any error messages.

#. Verify that the new store was created by querying the list of stores in the workspace:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores.xml

#. Verify that the new layer was published by querying the list of layers in the store:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores/borderstore/featuretypes.xml

   You can also view the layer in the GeoServer Layer Preview.

While this upload process is simple to execute, it is always recommended that your data be loaded into a spatial database first and then published by GeoServer.  See the next section for details on how to do this.


Publishing a PostGIS table in GeoServer
---------------------------------------

In this example, the shapefile or other dataset is assumed to be already loaded as a table in a PostGIS database.  The next step is to link the store to GeoServer and then publish a layer contained within.

.. note:: See the sections on :ref:`dataadmin.loading.shp2pgsql` or :ref:`dataadmin.loading.pgshapeloader` for ways to load a shapefile into PostGIS. 

#. First, a new store with the connection parameters to PostGIS needs to be established.  As the connection parameters are a bit verbose, it is easier if the content is saved into an XML file first rather than encoding everything in one long command.  So, create an XML file with the following content, substituting in the correct connection parameters:

   .. code-block:: xml

      <dataStore>
        <name>pgstore</name>
        <type>PostGIS</type>
        <enabled>true</enabled>
        <workspace>
          <name>opengeo</name>
          <atom:link xmlns:atom="http://www.w3.org/2005/Atom" rel="alternate" href="http://localhost:8080/geoserver/rest/workspaces/opengeo.xml" type="application/xml"/>
        </workspace>
        <connectionParameters>
          <entry key="port">54321</entry>
          <entry key="user">postgres</entry>
          <entry key="passwd">postgres</entry>
          <entry key="dbtype">postgis</entry>
          <entry key="host">localhost</entry>
          <entry key="database">OpenGeo</entry>
          <entry key="schema">public</entry>
        </connectionParameters>
        <__default>false</__default>
      </dataStore>

#. Save this file as :file:`pgrest.xml`.  The filename itself is not important and need not match the store name.

#. Load this content into GeoServer via the following command:

   .. code-block:: console

      curl -v -u admin:geoserver -X POST -H "Content-type: text/xml" -T pgrest.xml http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores.xml

   If the command was run successfully, you should see in the output:

   .. code-block:: console

      < HTTP/1.1 201 Created

   If you see a ``500 Internal Server Error`` or ``405 Method Not Allowed``, or any other error, then the command was issued incorrectly, and you should verify the syntax and content.  Examine the output and logs for any error messages.

#. The store created in the above example was called ``pgstore``, as that was the string contained in the ``<name>`` tag.  To verify that the store was created successfully, run the following command:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores/pgstore.xml

   .. note:: The password to this database, unencrypted in our example, is displayed encrypted here.


Now that the connection has been made, we can now publish a table from this database as a layer in GeoServer.

#. As the parameters involved in this publish command are very minimal, we don't need to save them as a separate file (although that can be done).  Instead, we include the parameters inline in the request:

   .. code-block:: console

      curl -v -u admin:geoserver -X POST -H "Content-type: text/xml" -d "<featureType><name>lakes</name></featureType>" http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores/pgstore/featuretypes

   Some notes about this command:

   * This creates a layer based on the table of the same name.  In this example, the table and layer are both named ``lakes``.
   * The layer is contained in the previously created ``pgstore`` as part of the ``opengeo`` workspace.
   * This example uses a POST request, not PUT, because we are creating a new featuretype resource.

#. As before, if the command was run successfully, you should see in the output:

   .. code-block:: console

      < HTTP/1.1 201 Created

   If not, examine the output for errors.

#. To verify that the layer was published, run the following command:

   .. code-block:: console

      curl -v -u admin:geoserver -X GET http://localhost:8080/geoserver/rest/workspaces/opengeo/datastores/pgstore/featuretypes.xml

   You should see the layer listed in the output.  You can also view the layer in the GeoServer Layer Preview.



