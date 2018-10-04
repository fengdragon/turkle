# Turkle #

Run a clone of Amazon's Mechanical **Turk** service in your **l**ocal
**e**nvironment.

This tool is meant to be used as a web service running locally on your network
or personal machine. It will load HIT template files generated by the Amazon
Mechanical Turk web GUI provided to requesters for creating HITs. Input CSV files are
also uploaded to create a HIT based on the template with each row of
values in the CSV file.

The results of the HITs completed by the workers can be exported in CSV files.


# Installation #

Turkle works with either python 2 or python 3.

## Dependencies ##

- Turkle depends on the packages listed in `requirements.txt`.
  If the packages are not already installed in your environment, and you have
  an internet connection, then you can run the following command to install
  the required Python packages:

  ```bash
  pip install -r requirements.txt
  ```

  Using a virtual environment has the advantage of keeping the dependencies
  for this project separate from other projects. The actual syntax depends
  on what virtual environment package you are using, but it should work like this:

  ```bash
  virtualenv venv
  source venv/bin/activate
  pip install -r requirements.txt
  ```

## Setup ##

After the dependencies have been installed, you create and initialize the db:

```bash
python manage.py migrate
python manage.py createsuperuser
```

## Configuration ##

# Usage

The administrator loads HITs for the worker(s) to complete.
There are example HITS in the `examples` directory.
A batch of HITs consists of an HTML template and corresponding CSV data files.

## Running the server ##

```bash
python manage.py runserver 0.0.0.0:8000
```

This runs the Django development web server on port 8000.

## Creating user accounts ##

### Using the admin UI
 * Login with a super user account
 * Select Manage Users from the navigation header
 * Click the `Add User` button, fill out the form and submit

### Using the scripts
The `add_user.py` script adds a single user. Run it with the `-h` option for details.

The `import_users.py` script reads a CSV file to add users to Turkle.
The file must be formatted like:
```
username1,password1
username2,password2
```

## Loading a batch of HITs ##

### Using the admin UI

TODO: documentation on using the admin UI for templates and CSVs.

### Using the scripts
With a template html file and a batch CSV file, use the
`publish_hits.py` script to add them to Turkle.
If you have already added the template, you must use the admin UI
to add additional batches of HITs.

## Downloading a batch of HITs ##

### Using the admin UI

TODO

### Using the scripts
The `download_results.py` script downloads all HITs that have been completed
into a directory that the user selects.

# Worker instructions

Use your web browser to visit [http://localhost:8000](http://localhost:8000)
Log in if you've been given user credentials.
Find your assigned set of HITs and click the "Accept next HIT" button.

# Production deployment

While Turkle can run with the default sqlite database and Django development web server,
for anything more than a few annotators, we recommend using a scalable database and
a production quality WSGI HTTP server.

To run on port 80 of your server, a common configuration would be a web server like
Apache or nginx as a proxy server with a Python HTTP server like Gunicorn behind it.

## MySQL

First, you need to install the python mysqlclient library:

```bash
pip install mysqlclient
```
Note that this requires development headers for mysql and python to be installed.
The method for installing these depends on your operating system.

Second, create a database and database user for Turkle. In the examples below,
we are calling the database `turkle` and the user `turkleuser` with the password `password`.
Log into the mysql client and run the following commands:

```sql
CREATE DATABASE turkle CHARACTER SET UTF8;
CREATE USER turkleuser@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON turkle.* TO turkleuser@localhost;
FLUSH PRIVILEGES;
```

Third, update Turkle's settings to point at this database:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'turkle',
        'USER': 'turkleuser',
        'PASSWORD': 'password',
        'HOST': 'localhost'
    }
}
```
The last step is running the migration command to create the database tables.

## Running with Gunicorn

Gunicorn can be installed with pip:
```bash
pip install gunicorn
```

and run from Turkle's base directory like this:

```bash
gunicorn --bind 0.0.0.0:8000 turkle.wsgi
```

Common Gunicorn runtime options are available in its [Running Gunicorn documentation](http://docs.gunicorn.org/en/stable/run.html).

This serves Turkle's web pages on port 8000 but not the static files like CSS and JavaScript.
For serious production uses of Turkle, the best option is to use a proxy server
(like Apache or nginx) to serve the statis files. More details on that below.

If you don't want to set up a proxy server, you can use Whitenoise to serve
the static files. Install it:

```bash
pip install whitenoise
```

and then enable it in `turkle/settings.py` in the MIDDLEWARE section.

## Apache as a reverse proxy

To use Apache as the proxy server, enable proxying: `a2enmod proxy_http`.
Then edit the configuration of your site to include the proxy information:

```
ProxyRequests Off
ProxyPass /static/ !
ProxyPass / http://localhost:5000/
ProxyPassReverse / http://localhost:5000/
```

Apache will look in its default location for the static directory so you'll need to create
a symbolic link (more convenient) or copy the files (safer).
After starting Gunicorn and restarting Apache with the new configuration, you should
be able to view the site at http://localhost/ or whatever the appropriate host name.

Instructions for using Gunicorn with nginx are found on its [deploy page](http://docs.gunicorn.org/en/latest/deploy.html).
You will still need to configure nginx to serve the static files as we did with Apache.

# Docker usage

Instead of installing Turkle and dependencies directly, you can run Turkle as a Docker container, using scripts to manage your HIT templates and data.
Either build a Turkle image:

```bash
docker build --force-rm -t hltcoe/turkle .
```

or pull the latest from the Docker registry:

```bash
docker pull hltcoe/turkle
```

and start a container with an easy name, and mapping container port 8080 somewhere on the Docker host (e.g. 18080):

```bash
docker run -d --name container_name -p 18080:8080 hltcoe/turkle
```

The Docker container has an `admin` user with a default password of `admin`.

You can change the default admin password by:

- connecting to the exposed container port with a web browser (e.g. connecting to http://localhost:18080/)
- logging in with username `admin` and password `admin`
- clicking on the "Manage Users" link on the top left of the screen
- clicking on the "CHANGE PASSWORD" link at the top right of the screen
- filling out and submitting the Change Password form

Your annotator can now browse to that port on the Docker host.  To give them something to do, upload an Amazon Turk HIT template and data:

```bash
python scripts/publish_hits.py -u [superuser] --server localhost:18080 template.html data.csv
```

At any point, you can download the current state of annotations:

```bash
python scripts/download_results.py -u [superuser] --server localhost:18080
```
