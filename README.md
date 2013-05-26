===
DYE
===

UNSTABLE SOFTWARE
=================

*Please be aware that DYE is not stable yet - it is a rework of our internal
deployment scripts to make them easier to reuse, and the rework is a work in
progress.  Please have a play, test against your dev server and give us
feedback.  But don't download and immediately run against your production
server.*

DYE is a bacronym for "Deploy Your Environment" - a set of scripts and
functions to deploy your web app along with the required python libraries in a
virtualenv, either locally or on a remote server. It is built on fabric. It is
most well developed for Django web apps, but we have used it for PHP projects
aswell.

You should copy the example/deploy directory to a deploy/ directory in your
project and edit the project_settings.py file. You may also want to add
localtasks.py and/or localfab.py

You should then replace the manage.py in the django directory with the manage.py
from the examples/django/ directory.

Expected Project Structure
==========================

A bare bones project structure would be:

    /apache                    <- contains config files for apache, for different servers
        staging.conf
        production.conf
    /deploy                    <- contains settings and scripts for deployment
        bootstrap.py
        fab.py
        localfab.py            <- optional
        localtasks.py          <- optional
        pip_packages.txt       <- list of python packages to install
        project_settings.py
        tasks.py
        ve_mgr.py
    /django
        /django_project        <- top level for Django project
            manage.py          <- a modified version of manage.py - see examples/
            .ve/               <- contains the virtualenv
            local_settings.py  <- a link to the real local_settings.py.env
            local_settings.py.dev
            local_settings.py.staging
            local_settings.py.production
            manage.py          <- our modified version
            private_settings.py   <- generated by these scripts
            settings.py        <- this will import local_settings.py
            urls.py
    /wsgi                      <- holds WSGI handler
        wsgi_handler.py

A certain amount of the directory structure can be overridden in
project_settings.py but that is not well tested currently.

Tasks.py
========

tasks.py is designed to make it easy to get your development environment up and
running easily. Once the project is set up, getting going should only require:

    cd deploy
    ./bootstrap.py
    ./tasks.py deploy:dev
    cd ../django/django_project
    ./manage.py runserver

bootstrap.py will create the virtualenv and install the python packages required

tasks.py deploy:dev will:

* generate a private_settings.py (random database password and Django secret key)
* link to one of your local_settings files (selects database etc)
* init and update git submodules (if any)
* create the database (if using MySQL at least) and run syncdb and migrations

Your Django application will then be good to go.

tasks.py includes a number of other tasks that should make your life easier. It
also makes it easy to add your own tasks and integrate them into the deploy task
by using:

localtasks.py
-------------

This is a file where you can define your own functions to do stuff that you
need for your project. You can also override functions from tasklib.py simply
by defining a function with the same name as the function in tasklib.py

You can override the main `deploy()` function, but you might lose out if the
deploy function starts to do more.  Generally a better strategy is to define a
`post_deploy()` function - this will be called by dye if it exists.

manage.py
---------

We use a modified version of manage.py that knows about the virtualenv in the
`.ve/` directory. So when you run manage.py it will automatically relaunch itself
inside the virtualenv, so you don't have to worry about activating/deactivating.

It also knows about the list of packages in `pip_packages.txt` - if that is
updated without the virtualenv being updated (or if the virtualenv doesn't
exist) then `manage.py` will complain. You can then create/update the virtualenv
with:

    ./manage.py update_ve

Note that update_ve will only update the virtualenv when required. Though you
can use --force to do it anyway. Also note that update_ve will completely delete
the old virtualenv and recreate it from scratch. To just add a new package, you
can run:

   ./manage.py update_ve_quick

Fabric and DYE
==============

We have developed a set of fabric functions for our standard project layout.
In order to avoid violating the DRY principle, we delegate the work to tasks.py
where possible. Our standard fab deploy will:

* check if you have made any local changes to the server. If it finds any it
  will alert you to them and give you the choice of whether to continue or not.
* if using git it will check the branch set in project_settings.py, the branch
  currently on the server and the branch you are currently on locally.  If there
  is a mismatch it will ask you to confirm what branch you want to use.
* create a copy of the current deploy, and a database dump, so you can rollback
  easily to the last known state.
* unlink the webserver config and reload the web server (effectively turning off
  the site)
* create the directory on the server (if this is the first deploy)
* checkout or update the project from your repository (git, svn and CVS
  currently supported)
* ensure the virtualenv is created and packages installed (as bootstrap.py does)
* call tasks.py deploy
* relink the webserver config and reload the web server (effectively turning on
  the site again)

As with tasks.py you can add extra functions and override the default behaviour
by putting functions in:

localfab.py
-----------

Server Directory Structure
--------------------------

Dye will create /var/django/project_name and in that directory will be:

    dev/           <- contains the checked out code
    previous/      <- copies for rollback, with directories named by timestamp

You can override the project root with the server_home variable in
project_settings.py
