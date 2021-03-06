#mysqlapi

[![Build Status](https://secure.travis-ci.org/tsuru/mysqlapi.png?branch=master)](http://travis-ci.org/tsuru/mysqlapi)

This is a service API for MySQL, used for [tsuru](https://github.com/tsuru/tsuru).


Installation
------------

In order to have mysql API ready to receive requests, we need some bootstrap stuff.

The first step is to install the dependencies. Let's use pip to do it:

    $ pip install -r requirements.txt

Now we need to run syncdb:

    $ python manage.py syncdb

Exporting enviroment variable to set the settings location:

    $ export DJANGO_SETTINGS_MODULE=mysqlapi.settings


Choose your configuration mode
------------------------------

There are three modes to configure the API usage behavior:

- `shared`: this configuration forces all applications to share the same mysql
  installation, in this mode, mysql API will create a new user and a new
  database when added/binded by an app.
- `dedicated (on-demmand)`: every app using mysql will have a single vm for
  it's usage, in this mode, mysql API will create a vm, install everything
  needed to run mysql based on a predefined AMI and create a user and password.
- `dedicated (pre-provisioned)`: every app using mysql will have a single MySQL
  instance, pre-provisioned.

Everything that is needed by the application to connect with mysql is provided
automatically by tsuru, using environment variables, e.g. when you add/bind
your app with mysql service, tsuru will export all environment variables
returned by mysql API.


Shared Configuration
--------------------

To run the API in shared mode, is needed to have a mysql installed and export
two enviroment variables.

One variable is to set the mysql host. If the shared mysql database is
installed in the sabe vm that the app is, you can use `localhost` for
``MYSQLAPI_SHARED_SERVER``, but you'll also need to set up a externally
accessible endpoint to be used by the apps that are using the service:

    $ MYSQLAPI_SHARED_SERVER=mysqlhost.com
    $ MYSQLAPI_SHARED_SERVER_PUBLIC_HOST=publichost.com


Running the api
---------------

    $ gunicorn wsgi -b 0.0.0.0:8888


Try your configuration
----------------------

You can try if the previous configuration worked using curl:

    $> curl -d 'name=myapp' http://youmysqlapi.com/resources

This call is the same as to ``tsuru service-add <service-name>
<service-instance-name>`` and will return 201 if everything goes ok.

If there are any problems, be welcome to report an issue :)


Install as application
----------------------

You can deploy `mysqlapi` as tsuru appplication.


### Install MySQL server (Debian/Ubuntu)

First you should have a MySQL server. In Debian/Ubuntu, use `apt-get` to install it.

```bash
$ sudo apt-get install mysql-server-5.6
```

During install the installation script will aks you the password for `root` user.


#### Create database for mysqlapi

After install MySQL, you need to create a user and a database for `mysqlapi`,
that is needed to store informations about created instances.

```
mysql -u root -p
```

Here is an example:

```sql
CREATE DATABASE mysqlapi CHARACTER SET utf8 COLLATE utf8_bin;
GRANT ALL PRIVILEGES ON mysqlapi.* TO 'mysqlapi'@'%' IDENTIFIED BY 'mysqlpass';
```
Configure mysql to accept external connection, create a file `/etc/mysql/conf.d/bind.cnf` with the following content:

```
[mysqld]
bind-address = 0.0.0.0
```

To finish restart MySQL server:

```
sudo service mysql restart
```


### Install service

Now you can install `mysqlapi` service. In your tsuru client machine (with crane installed):

```bash
$ git clone https://github.com/tsuru/mysqlapi
$ tsuru app-create mysqlapi python
```

Export these environment variables:

```bash
# mysqlapi's database configure
$ tsuru env-set -a mysqlapi MYSQLAPI_DB_NAME=mysqlapi
$ tsuru env-set -a mysqlapi MYSQLAPI_DB_USER=mysqlapi
$ tsuru env-set -a mysqlapi MYSQLAPI_DB_PASSWORD=mysqlpass
$ tsuru env-set -a mysqlapi MYSQLAPI_DB_HOST=db.192.168.50.4.nip.io

# salt used to hash the username/password
$ tsuru env-set -a mysqlapi MYSQLAPI_SALT=******

# Exporting enviroment variable to set the settings location
tsuru env-set -a mysqlapi DJANGO_SETTINGS_MODULE=mysqlapi.settings
```

Export these variables to specify the shared cluster:

```bash
# these settings can be different with mysqlapi's database
$ tsuru env-set -a mysqlapi MYSQLAPI_SHARED_SERVER=db.192.168.50.4.nip.io
$ tsuru env-set -a mysqlapi MYSQLAPI_SHARED_SERVER_PUBLIC_HOST=db.192.168.50.4.nip.io
$ tsuru env-set -a mysqlapi MYSQLAPI_SHARED_PORT=3306
$ tsuru env-set -a mysqlapi MYSQLAPI_SHARED_USER=root
$ tsuru env-set -a mysqlapi MYSQLAPI_SHARED_PASSWORD=******
```

Configuration are finished now. Deploy the service.

```bash
# Find out git repository then deploy the service
$ cd mysqlapi
$ tsuru app-info -a mysqlapi | grep Repository
$ git remote add tsuru git@192.168.50.4.nip.io:mysqlapi.git
$ git push tsuru master
```

Configure the service template and point it to your application:

```bash
# you can find out production address from app-info
$ tsuru app-info -a mysqlapi | grep Address
# set production address
$ editor service.yaml
$ crane create service.yaml
```

To list your services:

```bash
$ crane list
# OR
$ tsuru service-list
```
