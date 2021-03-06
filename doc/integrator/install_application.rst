.. _integrator_install_application:

Install an existing application
===============================

On this page we explain all the procedures to build an application from
only the code.

For example If you want to use an existing database you should ignore
all the commands concerning the database.

This guide considers that:
 - We use a server manages by Camptocamp, meaning:
    - all dependencies described in the
      :ref:`requirements <integrator_requirements>` are installed,
    - Postgres has a GIS template ``template_postgis`` and a user ``www-data``,
    - Apache uses the user ``www-data``.
 - Use Git as revision control.

For the others system there is some notes to give some help.

Set up the database
-------------------

Any c2cgeoportal application requires a PostgreSQL/PostGIS database. The
application works with its own tables, which store users, layers, etc. These
tables are located in a specific schema of the database.

.. note::

    Multiple specific schemas are actually used in a parent/child architecture.

If the application has MapServer layers linked to PostGIS tables, these tables
and the application-specific tables must be in the same database, preferably in
separate schemas. This is required for layer access control (*restricted
layers*), where joining user/role tables to PostGIS layer tables is necessary.

.. _integrator_install_application_create_database:

Create the database
~~~~~~~~~~~~~~~~~~~

To create the database you can use:

.. prompt:: bash

    sudo -u postgres createdb <db_name> -T template_postgis

with ``<db_name>`` replaced by the actual database name.

.. note::

   if you don't have the template_postgis you can use:

   with Postgres >= 9.1 and PostGIS >= 2.1:

   .. prompt:: bash

       sudo -u postgres createdb -E UTF8 -T template0 <db_name>
       sudo -u postgres psql -c "CREATE EXTENSION postgis;" <db_name>

   with older versions:

   .. prompt:: bash

       sudo -u postgres createdb -E UTF8 -T template0 <db_name>
       sudo -u postgres psql -d <db_name> -f /usr/share/postgresql/9.1/contrib/postgis-2.1/postgis.sql
       sudo -u postgres psql -d <db_name> -f /usr/share/postgresql/9.1/contrib/postgis-2.1/spatial_ref_sys.sql

   Note that the path of the postgis scripts and the template name can
   differ on your host.

.. _integrator_install_application_create_schema:

Create the schema
~~~~~~~~~~~~~~~~~

Each parent or child needs two application-specific schemas,
then to create it use:

.. prompt:: bash

    sudo -u postgres psql -c "CREATE SCHEMA <schema_name>;" <db_name>
    sudo -u postgres psql -c "CREATE SCHEMA <schema_name>_static;" <db_name>

with ``<db_name>`` and ``<schema_name>`` replaced by the actual database name,
and schema name ('main' by default), respectively.

.. _integrator_install_application_create_user:

Create a database user
~~~~~~~~~~~~~~~~~~~~~~

We use a specific user for the application, ``www-data`` by default.

.. note::

   It the user doesn't already exist in your database, create it first:

   .. prompt:: bash

        sudo -u postgres createuser -P <db_user>

Give the rights to the user:

.. prompt:: bash

    sudo -u postgres psql -c 'GRANT SELECT ON TABLE spatial_ref_sys TO "www-data"' <db_name>
    sudo -u postgres psql -c 'GRANT ALL ON TABLE geometry_columns TO "www-data"' <db_name>
    sudo -u postgres psql -c 'GRANT ALL ON SCHEMA <schema_name> TO "www-data"' <db_name>
    sudo -u postgres psql -c 'GRANT ALL ON SCHEMA <schema_name>_static TO "www-data"' <db_name>

.. note::

   If you don't use the www-data user for Apache replace it by the right user.


Install the application
-----------------------

Get the application source tree
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If Git is used for the application use the following command to get the
application source tree:

.. prompt:: bash

    git clone git@github.com:camptocamp/<my_project>.git <my_project>

c2cgeoportal applications include a Git submodule for CGXP. The following
commands should be used to download CGXP and its dependencies:

.. prompt:: bash

    cd <my_project>
    git submodule update --init
    git submodule foreach git submodule update --init

The ``foreach`` command aims to init and update CGXP's own submodules, for GXP,
OpenLayers and GeoExt.

.. note::

    We don't just use ``git submodule update --init --recursive`` here because
    that would also download GXP's submodules. We don't want that because we
    don't need GXP's submodules. CGXP indeed has its own submodules for
    OpenLayers and GeoExt.

.. important::

    If you want other people than you to be able to run ``make`` from an
    application clone created by you then you need to change the application
    directory's permissions using ``chmod -R g+w``.  You certainly want to do
    that if the application has been cloned in a shared directory like
    ``/var/www/<vhost>/private``.

Non Apt/Dpkg based OS Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Disable the package checking:

In the ``<package>.mk`` add::

    TEST_PACKAGES = FALSE

Windows Specific Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some changes in the apache wsgi and mapserver configurations are required to make
c2cgeoportal work on Windows.

apache/wsgi.conf.mako
^^^^^^^^^^^^^^^^^^^^^

``WSGIDaemonProcess`` and ``WSGIProcessGroup`` are not supported on windows.

(`WSGIDaemonProcess ConfigurationDirective
<http://code.google.com/p/modwsgi/wiki/ConfigurationDirectives#WSGIDaemonProcess>`_
"Note that the ``WSGIDaemonProcess`` directive and corresponding features are not
available on Windows or when running Apache 1.3.")

The following lines must be commented/removed::

    WSGIDaemonProcess c2cgeoportal:${instanceid} display-name=%{GROUP} user=${modwsgi_user}
    ...
    WSGIProcessGroup c2cgeoportal:${instanceid}

apache/mapserver.conf.mako
^^^^^^^^^^^^^^^^^^^^^^^^^^

The path to Mapserver executable must be modified::

    ScriptAlias /${instanceid}/mapserv C:/path/to/ms4w/Apache/cgi-bin/mapserv.exe

mapserver/c2cgeoportal.map.mako
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You must specify the path to the mapserver's epsg file by uncommenting and adapting
this line under ``MAP`` (use regular slash ``/``) ::

    PROJ_LIB" "C:/PATH/TO/ms4w/proj/nad"


RHEL 6 Specific Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Specific settings are required when the c2cgeoportal application is to be run
on RedHat Enterprise Linux (RHEL) 6.

.. note::

    First of all, note that, with RHEL, you cannot install the c2cgeoportal
    application in your homedir. If you do so, you will get the following error
    in the Apache logs::

        (13)Permission denied: access to /~elemoine/ denied

    So always install the application in an Apache-accessible directory. On
    Camptocamp *puppetized* servers you will typically install the application
    in ``/var/www/vhosts/<vhost>/private/dev/<username>/``, where ``<vhost>``
    is the name of the Apache virtual host, and ``<username>`` is your Unix
    login name.

vars_<project>.xaml
^^^^^^^^^^^^^^^^^^^

By default, ``mod_wsgi`` processes are executed under the ``www-data`` Unix
user, which is the Apache user. In RHEL 6, there's no user ``www-data``, and
the Apache user is ``apache``. To accomodate that edit ``vars_<project>.yaml`` and
set ``modwsgi_user`` to ``apache`` in the ``[vars]`` section::

    [vars]
    ...
    modwsgi_user = apache


Also, by default, the path to Tomcat's ``webapps`` directory is
``/srv/tomcat/tomcat1/webapps``. On RHEL 6, Tomcat is located in
``/var/lib/tomcat6/``. To accomodate that the ``output`` path of the
``[print-war]`` part should be changed::

    [print-war]
    output = /var/lib/tomcat6/webapps/print-c2cgeoportal-${instanceid}.war

apache/mapserver.conf.mako
^^^^^^^^^^^^^^^^^^^^^^^^^^

On RHEL 6 the ``mapserv`` binary is located in ``/usr/libexec/``. The
``mapserver.conf.mako`` Apache config file assumes that ``mapserv`` is located in
``/usr/lib/cgi-bin/``, and should therefore be changed::

    ScriptAlias /${instanceid}/mapserv /usr/libexec/mapserv

apache2ctl
~~~~~~~~~~

On RedHat the commands hasn't the '2'!
Then to graceful apache do::

    /usr/sbin/apachectl graceful

.. _integrator_install_application_install_application:

Install the application
~~~~~~~~~~~~~~~~~~~~~~~

If it doesn't already exist, create a ``<user>.mk`` file
(where ``<user>`` is for example your username),
that will contain your application special
configuration:

.. code:: make

    INSTANCE_ID = <instanceid>
    DEVELOPMENT = TRUE

    include <package>.mk

.. note::

    The ``<instanceid>`` should be unique on the server, the username is a good
    choice or something like ``<user>-<sub-project>`` in case of parent/children project.

Add it to Git:

.. prompt:: bash

    git add <user>.mk
    git commit -m "Add user build file"

Then you can build and install the application with the command:

.. prompt:: bash

    make -f <user>.mk build

This previous command will do many things like:

  * download and install the project dependencies,

  * adapt the application configuration to your environment,

  * build the javascript and css resources into compressed files,

  * compile the translation files.

Once the application is built and installed, you now have to create and
populate the application tables, and directly set the version (details later):

.. prompt:: bash

    .build/venv/bin/alembic upgrade head
    .build/venv/bin/alembic -c alembic_static.ini upgrade head

Your application is now fully set up and the last thing to do is to configure
apache so that it will serve your WSGI c2cgeoportal application. So you just
have to include the application apache configuration available in the
``apache`` directory. On servers managed by Camptocamp, add a ``.conf`` file in
``/var/www[/vhost]/<vhostname>/conf/`` (``[/vhost]`` means that the vhost folder
is optional, ``<vhostname>`` is a folder that should already exist (created by
the system administrator), that corresponds to the virtual host)
with the following content::

    Include /<project_path>/apache/*.conf

where ``<project_path>`` is the path to your project.

Reload apache configuration and you're done:

.. prompt:: bash

    sudo /usr/sbin/apache2ctl graceful

Your application should be available at:
``http://<hostname>/<instanceid>``.

Where the ``<hostname>`` is directly linked to the virtual host,
and the ``<instanceid>`` is the value you provided before.
