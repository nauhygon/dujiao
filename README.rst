.. To convert this document to HTML
.. rst2html.py --stylesheet=voidspace.css,transition-stars.css SETUP.rst > SETUP.html

=================================================================
Deploy Django with Nginx/Gunicorn/Virtualenv/Supervisor on Ubuntu
=================================================================

--------------------
A step-by-step guide
--------------------

:Author: Oak Nauhygon
:Contact: nauhygon [at] gmail.com
:Version: 0.0.1
:Copyright: Public domain

.. contents:: Table of Contents


Create a Django project using Pinax
===================================

Install build/setup tools
-------------------------

Steps::

   $ sudo apt-get install build-essential python-dev python-setuptools
   $ sudo apt-get build-dep python-imaging
   $ sudo easy_isntall virtualenv

Prepare a working directory
---------------------------

Steps (sub /path/to/work with a valid one)::

   $ mkdir /path/to/work
   $ export MYWORK=/path/to/mywork

Set up a virtualenv
-------------------

Steps::

   $ cd $MYWORK
   $ virtualenv --no-site-packages --distribute myenv
   $ cd myenv
   $ . bin/activate


Use Pinax to crate a project
----------------------------

Steps::

   $ pip install pinax yolk
   $ pinax-admin setup_project myproj

   $ cd myproj
   $ python manage.py syncdb
   $ python manage.py runserver 0.0.0.0:8000
   # check http://<hostname>:8000

Deploy project with Nginx and Gunicorn
======================================

Run project with Gunicorn
-------------------------

Steps::

   $ pip install gunicorn
   # Add gunicorn to INSTALLED_APPS in myproj/settings.py
      INSTALLED_APPS = {
           # ......
           gunicorn
           # ......
      }
   $ python manage.py run_gunicorn --workers=2
   # check http://localhost:8000 on localhost
   # use "--bind 0.0.0.0:8000" to check http://<hostname>:8000 on a different machine

Set up ngnix
------------

Steps::

    $ sudo apt-get install nginx
    $ sudo /etc/init.d/nginx start
    # edit /var/www/ngnix-default/index.html and check http://<hostname>

    # Insert below into /etc/nginx/sites-enabled/default

        upstream app_server_djangoapp {
           server localhost:8000 fail_timeout=0;
        }
        server {
           # ......
           location / {
               #root   /var/www/nginx-default;
               #index  index.html index.htm;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header Host $http_host;
               proxy_redirect off;
               if (!-f $request_filename) {
                   proxy_pass http://app_server_djangoapp;
                   break;
              }
           }
	   # ......
        }

    $ sudo /etc/init.d/nginx restart
    # check http://<hostname> (note now on regular port 80 instead of 8000!)

Use Supervisor to control/monitor Gunicorn server
=================================================

Install Supervisor
------------------

Steps::

   $ sudo apt-get install supervisor

Configure Supervisor
--------------------

Steps::

    $ cd $MYWORK/myenv/myproj/deploy

    $ cat > run_gunicorn

    #!/bin/bash

    # A script to run myproj using Gunicorn

    dir0=`pwd`

    # get script dir
    script=$0;
    script_dir=`dirname $script`

    # assuming dir structure: <venv>/<project>/deploy
    venv=$script_dir/../..
    proj=$script_dir/..

    # start virtualenv
    . $venv/bin/activate

    # start server
    cd $proj
    python manage.py run_gunicorn --workers=2

    # end virtualenv
    deactivate

    cd $dir0
    ^D
    
    $ chmod +x run_gunicorn

    $ cat > djangoapp.conf
    [program:djangoapp]
    command=/path/to/mywork/myenv/myproj/deploy/run_gunicorn
    directory=/path/to/mywork/myenv/myproj
    user=www-data
    autostart=true
    autorestart=true
    redirect_stderr=True
    ^D

    $ cd /etc/supervisor/conf.d
    $ sudo ln -s /path/to/mywork/myenv/myproj/deploy/djangoapp.conf

    $ sudo /etc/init.d/supervisor start
    $ sudo supervisorctl
    djangoapp RUNNING pid 953, uptime 0:01:01

