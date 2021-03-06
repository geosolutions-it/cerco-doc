.. _install_ckan:

###################
Installing CKAN 2.6
###################

============
Introduction
============

In this document you'll only find specific information for installing CKAN, some required ancillary applications 
and some ufficial CKAN extensions.

It is expected that the base system has already been properly installed and configured as described in previous pages such as  
:ref:`os_tomcat_install`, :ref:`os_postgres_install`, :ref:`os_httpd_install`, :ref:`install_solr`.

In such documents there are information about how to install the required base components.


=================
Required packages
=================

Install the software packages needed by CKAN::

   yum install gcc gcc-c++ make git gdal geos
   yum install libxml2 libxml2-devel libxslt libxslt-devel    
   yum install gdal-python python-pip python-imaging  python-virtualenv \
               libxml2-python libxslt-python python-lxml \
               python-devel python-babel python-psycopg2 \
               python-pylons python-repoze-who python-repoze-who-plugins-sa \
               python-repoze-who-testutil python-repoze-who-friendlyform \
               python-tempita python-zope-interface policycoreutils-python 


====================
Creating a CKAN user
====================
 
The ``ckan`` user is created with a shell of ``/sbin/nologin`` and a home directory of ``/usr/lib/ckan``::

   useradd -m -s /sbin/nologin -d /usr/lib/ckan -c "CKAN User" ckan

Should you need to run anything as user ``ckan``, you can switch to the ckan account
by issuing this command as ``root`` ::
   
   su -s /bin/bash - ckan


==============
Setup CKAN dir
==============

Open the ckan home directory up for read access so that the content 
will eventually be able to be served out via httpd ::

   chmod 755 /usr/lib/ckan

Under CentOS and RedHay you may have to modify the defaults and the current file context of the newly created directory 
such that it is able to be served out via httpd ::

   semanage fcontext --add --ftype -- --type httpd_sys_content_t "/usr/local/ckan(/.*)?"
   semanage fcontext --add --ftype -d --type httpd_sys_content_t "/usr/local/ckan(/.*)?"
   restorecon -vR /usr/lib/ckan

.. _ckan_db_setup:
    
========================
PostgreSQL configuration
========================

Create the ``ckan`` user in postgres::

   su - postgres -c "createuser -S -D -R -P ckan"
   
and annotate the password for such user.
As an example, we'll use ``ckan_pw`` to show where this info will be needed.

Create the ckan db::

   su - postgres -c "createdb -O ckan ckan -E utf-8"


============================
Configuring CKAN environment
============================


Installing python dependencies
------------------------------

As user ``root`` run::

   easy_install pip
   pip install virtualenv


As user ``ckan``, go to ckan home dir::

   cd
   
Create a virtualenv called ``default``::

   virtualenv --no-site-packages default
   
Activate the vitualenv::
   
   . default/bin/activate
   
Download and install CKAN::
   
   pip install -e 'git+https://github.com/ckan/ckan.git@release-v2.6-latest#egg=ckan'

   
Enable the path for some postgres utilities::   
   
   export PATH=$PATH:/usr/pgsql-9.6/bin/
   

There are some missing requirements in CKAN 2.6, so you have to run::

   pip install setuptools==36.1 
   pip install pytz


Download and install the necessary Python modules to run CKAN into the isolated Python environment::
 
   pip install -r default/src/ckan/requirements.txt
   
  
.. _install_ckan_solr_conf:

Solr configuration
------------------

If solr is running, stop it::
 
   systemctl stop tomcat@solr

Configure in Solr the CKAN schema::

   cd /etc/solr/ckan/conf/ 
   mv schema.xml schema.xml.original
   ln -s /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /etc/solr/ckan/conf/schema.xml   
   chown tomcat: schema.xml

(Re)start solr::
   
   systemctl start tomcat@solr

Perform a test call to find out if Solr is running properly::
   
       curl -i http://localhost:8081/solr/ | less
   
If you get a ``404`` error probably Solr has some problems.
   
You should check the file ``/var/lib/tomcat/solr/logs/localhost.DATE.log`` for any error.
   

.. note::   
   Should Solr complain about missing libs, copy them from the dist directory::   

      systemctl stop tomcat@solr
      cp -v /root/download/solr-4.5.0/dist/solrj-lib/* /var/lib/tomcat/solr/webapps/solr/WEB-INF/lib/
      systemctl start tomcat@solr

.. important::   
   Note that solr requires the current hostname to be bound to a real IP address.

   This is an example of a hostname not properly bound::   

     [root@ckan conf]# hostname 
     ckan
     [root@ckan conf]# ping ckan
     ping: unknown host ckan
     [root@ckan conf]#
   
   You'll have to edit the ``/etc/hosts`` file and add a line like this::
   
     10.10.100.70 ckan

   
.. _install_ckan_ckan_conf:
   
CKAN configuration
------------------

Create a default configuration file. 

As ``root`` create the directory ::

   mkdir /etc/ckan
   chown ckan: /etc/ckan/

As user ``ckan``, enter the *virtualenv* ::

   $ . /usr/lib/ckan/default/bin/activate
   (pyenv)$ paster make-config ckan /etc/ckan/default/production.ini 
   

Edit the file ``/etc/ckan/default/production.ini`` 

- DB connection parameters ::

   sqlalchemy.url = postgresql://ckan:PASSWORD@localhost/ckan
   solr_url = http://127.0.0.1:8081/solr/ckan-schema-2.0
    
- Site data ::

    ckan.site_id:
    ckan.site_title:
    ckan.site_url:

- Filesystem data location ::
    
    ckan.storage_path = /var/lib/ckan/storage

- Mail notifications (es.) ::

    email_to = info@the.project.org
    smtp_server = server.smtp.for.the.project.org
    error_email_from = notifications@project.org

- Language ::

    ckan.locale_default = it
    ckan.locales_offered = it en 
    ckan.locale_order = it en


The file ``who.ini`` (the *Repoze.who* configuration file) needs to be accessible 
in the same directory as your CKAN config file, so create a symlink to it::

    ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini


Directories init
''''''''''''''''

As  ``root``::
  
   mkdir /var/log/ckan
   chown ckan: /var/log/ckan

   mkdir -p /var/lib/ckan/storage
   chown ckan: -R /var/lib/ckan

   
DB init
'''''''

.. note::
   In case you need to restore an existing DB, or you are migrating an older CKAN deploy,
   please skip this section and follow the steps :ref:`install_ckan_db_migration` instead.     

As user ``ckan``::

   . default/bin/activate
   paster --plugin=ckan db init -c /etc/ckan/default/production.ini

.. note::
   The ``db init`` procedure needs solr to be running.

.. _install_ckan_db_migration:

DB migration
''''''''''''

In case you are restoring an existing DB, or you are migrating an older CKAN deploy, you don't have to init your DB from scratch
and create initial users, since all the existing data will be restored into the new CKAN instance.

As ``postgres`` user, restore the dumped DB data::

   psql -f FULL_PATH_TO_DUMP_FILE ckan

Edit the ``/etc/ckan/default/production.ini`` file and comment out the ``ckan.plugins`` line
by putting a ``#`` comment symbol at the beginning::   
   
   # ckan.plugins = [...] 
   
As ``ckan`` user, activate the virtualenv and perform the migration::

   . /usr/lib/ckan/default/bin/activate
   paster --plugin=ckan db upgrade    --config=/etc/ckan/default/production.ini
   
Uncomment the ``ckan.plugins`` line in ``/etc/ckan/default/production.ini``.

Reindex the dataset::

   paster --plugin=ckan search-index rebuild   --config=/etc/ckan/default/production.ini


CKAN users
''''''''''

.. note::
   In case you need to restore an existing DB, or you are migrating an older CKAN deploy,
   a sysadmin should already be configured, so you may skip this section.     

Add a user with sysadmin privileges using this command ::

   (pyenv)$ paster --plugin=ckan sysadmin add USERNAME -c /etc/ckan/default/production.ini
   

Test  CKAN
''''''''''

Run CKAN as user ``ckan``::

   (pyenv)$ paster serve /etc/ckan/default/production.ini &
   

==========================
Apache httpd configuration
==========================

As ``root``, create the file ``/etc/httpd/conf.d/92-ckan.conf`` and add the following content::

   <VirtualHost *:80>
      ProxyPass        / http://localhost:5000/
      ProxyPassReverse / http://localhost:5000/
   </VirtualHost>

and reload the configuration ::

   systemctl reload httpd
   

SElinux
-------

`httpd` is blocked by default by SELinux so that it can't establish internal TCP connections; 
in order to allow http proxying, issue the following command ::

   setsebool -P httpd_can_network_connect 1

.. _install_supervisord_ckan:

=========================
supervisord configuration
=========================

CKAN does not provide a default script for autostarting; we'll use the *supervisord* daemon to do that.

As root::

   yum install supervisor
   systemctl enable supervisord

Create the file ``/etc/supervisord.d/ckan.ini`` and add the following lines to handle CKAN::

   [program:ckan]
   command=/usr/lib/ckan/default/bin/paster serve /etc/ckan/default/production.ini
   user=ckan
   autostart=true
   autorestart=true
   numprocs=1
   log_stdout=true
   log_stderr=true
   stdout_logfile=/var/log/ckan/out.log
   stderr_logfile=/var/log/ckan/err.log
   logfile=/var/log/ckan/ckan.log
   startsecs=10
   startretries=3

Run supervisord::

   systemctl start supervisord

