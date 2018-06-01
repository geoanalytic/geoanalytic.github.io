---
layout: post
comments: true
title: A reference server for geopaparazzi cloud profiles - Part 2
---

Deployment for production.  In a [previous post](https://geoanalytic.github.io/a-reference-server-for-geopaparazzi-cloud-profiles/), I described how to set up a local instance of a server for [Geopaparazzi](https://github.com/geopaparazzi/geopaparazzi) cloud profiles.  The goal of this work is to build a robust scaleable server that can provide security, spatial data management, and a quality user interface.  Following the lead of developers who know more than I do, it is built as a [Docker-Compose](https://docs.docker.com/compose/) recipe using [PostGIS](https://postgis.net/), [Django](https://www.djangoproject.com/), [Caddy](https://caddyserver.com/) and several optional services.      

At the end of the previous post, we had a server running on a local network that we could use to upload/download the various files needed by Geopaparazzi on our mobile device.  The next step is to deploy our system using the `production.yml` recipe and arrange all the supporting services required to get our server on the internet.     

# What Are Cloud Profiles Anyway?          

Before getting started, we should consider what sort of data we are dealing with.  Reviewing the [documention for cloud profiles](http://geopaparazzi.github.io/geopaparazzi/#_geopaparazzi_cloud_server), there are some descriptive parameters plus a number of supporting files, including:    

* a Project file - this is a Geopaparazzi (SQLite) database (may be empty)    
* a Tags file - this is a form definition stored as JSON     
* one or more Basemaps - these could be raster, vector or tiled?     
* one or more Spatialitedbs - including a list of visible layers     
* one or more Otherfiles - could be images, videos, PDFs....      

Some of these files are fairly small while others could be quite large.  Many have a geospatial component, as does the `mapView` profile parameter.  Finally, the Geopaparazzi user has the ability to create new data that can be uploaded to the server and distributed to other users.  How to manage these complexities and capitalize on the power they provide underlies the true motivation for this project.    

# Prerequisites    

Before we can deploy our server to the wide world, there are a few things that will be required, including:    

* A suitable server with internet connection        
* A web domain that you control, and can create DNS entries for   

If you are already operating a web server on your premises, you probably already have these assets under control and can skip the first couple of steps discussed below.  If not, you can register a domain at [GoDaddy](https://www.godaddy.com) or one of their many competitors.  As for the server, the modern approach is to rent a cloud server.  There are some good low cost options available from [Google](https://cloud.google.com/) and [Amazon](https://aws.amazon.com/), but I am very happy with the quality of service at [Digital Ocean](https://www.digitalocean.com/), which has a much simpler interface, albeit with a narrower range of product offerings.     

> If you use this [Digital Ocean referral link](https://m.do.co/c/07e7a94179df) to sign up, you will recieve a \$10 credit, which is enough to run a capable server for 2 months.  Most of the setup instructions that follow assume the use of the lowest cost Digital Ocean server.    

# Step 1 - Spin Up A Server      

As discussed in the first post, there are a few options for how to deploy the reference server.  For now, I'll discuss using Docker on a cloud server.  Examples will be provided for Digital Ocean, but I'll make references to Google Cloud and AWS where I can.    

On Digital Ocean, you can create a new Droplet that has everything you need with a couple of mouse clicks.  Simply select [the One-Click Docker on Ubuntu](https://www.digitalocean.com/products/one-click-apps/docker/) option.  The lowest cost ($5/month) option is more than adequate.  When your droplet is ready, you should update the DNS records to point your domain (or at least a subdomain) to your new droplet.  [Complete instructions are provided here.](https://www.digitalocean.com/community/tutorials/an-introduction-to-digitalocean-dns)    

The one-click droplet has everything we need to get started.  If you are using some other service, you may need to install [Docker](https://docs.docker.com/install/) and [Docker-compose](https://docs.docker.com/compose/install/).     

# Step 2 - Set Up The Services    

This process is discussed in more detail in the previous post.  Briefly, install [cookiecutter](https://cookiecutter.readthedocs.io/en/latest/)    

```
$ pip install "cookiecutter>=1.4.0"
```

Now run it against this repo:    

```
$ cookiecutter https://github.com/geoanalytic/cookiecutter-geopaparazzi-server
```

You will be asked for some values, refer to the previous post for more detail.  The important things right now:   

* use docker and whitenoise   
* do not use mailhog, sentry, heroku, or travisci    
* replace any references to David Currie, trailstewards.com, etc with your own information.    

# Step 3 - Edit Some Configuration Settings    

Now cd into the directory that was just created for you and have a look at the contents.  We will need to edit a few configuration files.  These files are found in several locations that you should be aware of, although most should be fine with the default values.  Some important details are located at:    

| Category | File/Locations |    
|----------|----------------|
|Docker Config | production.yml, /compose folders|
|Django Config | /config and /config/settings folders|
|Site specific stuff | /.envs folders|

A key feature of the [Cookiecutter-Django](https://cookiecutter-django.readthedocs.io/en/latest/) approach is that values important to security and inter-container coordination are stored as environment variables and injected when the container is started.  These values can be found in the .envs directory which should look like this:   

```
.envs
├── .local
│   ├── .django
│   └── .postgres
└── .production
    ├── .caddy
    ├── .django
    └── .postgres
```

Note the file and folder names all begin with a dot (.) which prevents them from showing up on a default `ls` command or from being included when running git.    

## Email Setup    

The [django-allauth](https://django-allauth.readthedocs.io/en/latest/) application provides a number of useful functions for allowing users to register on our site, including using social accounts (Facebook, Google, etc...) to provide one click authentication.  The default sign-up procedure requires a prospective user to provide an email address which they then have to authenticate by clicking on a provided link.     
This feature requires the server to be able to send emails.  The default email server is [Mailgun](https://www.mailgun.com/) which allows you to send up to 10,000 emails per month at the free service level.  To use this service, sign up and get an access key.  You will need to create some DNS records as well.  [The instructions are here.](https://help.mailgun.com/hc/en-us/articles/202052074-How-do-I-verify-my-domain-)  Finally, edit the `.envs/.production/.django` configuration file and fill in the `MAILGUN_API_KEY` and `MAILGUN_DOMAIN` values.  You can also configure the `DJANGO_SERVER_EMAIL` setting if you like.  For example, this is what my settings look like:    

```
# .django
# ...
# Email
# ------------------------------------------------------------------------------
MAILGUN_API_KEY=key-968064b62df797e6287lJsd8L0d4e717s9
DJANGO_SERVER_EMAIL=info@trailstewards.com
MAILGUN_DOMAIN=mg.trailstewards.com
```

If you don't want to use Mailgun, you can fall back to the Sendmail service (this needs to be tested) by editing the `config/settings/production.py` file and commenting out the following two lines:    

```
# config/settings/production.py
# ...
# COMMENT OUT THE NEXT TWO LINES TO USE THE DJANGO EMAIL BACKEND
INSTALLED_APPS += ['anymail']  # noqa F405
EMAIL_BACKEND = 'anymail.backends.mailgun.EmailBackend'
```

This approach may require some additional setup to ensure that your emails are accepted by other servers.   

## Hosting Media Files    

We use the term _media files_ to refer to the downloadable (and uploadable) files that our server provides to the user application, in this case all the Spatialite DBs, Basemaps, etc.  The production configuration is set up to use an Amazon scaleable storage service called [S3](https://aws.amazon.com/s3/), or equivalent.  This is a good way to go, as it allows us to host large amounts of data and serve them from a high performance resource without having to allocate large amounts of disk space on our own server.    

To use S3, you will need an [Amazon AWS](https://portal.aws.amazon.com/billing/signup?nc2=h_ct&redirect_url=https%3A%2F%2Faws.amazon.com%2Fregistration-confirmation#/start) account.  Go to your S3 console, click the Create Bucket button and follow the instructions.  Then create an IAM user and get the credentials, AWS_ACCESS_KEY and AWS_SECRET_ACCESS_KEY.  [Here are some step by step instructions.](https://www.cloudberrylab.com/blog/how-to-find-your-aws-access-key-id-and-secret-access-key-and-register-with-cloudberry-s3-explorer/).  Finally, edit the `AWS_*` settings in the `.envs/.production/.django` file.  Here is what it should look like:      

```
# .django
# AWS
# ------------------------------------------------------------------------------
DJANGO_AWS_ACCESS_KEY_ID=ASLKFJASLK0WEJ9
DJANGO_AWS_SECRET_ACCESS_KEY=lkj09LSKJDFOSIFLKalkjf0-9SLKJJFOEI
DJANGO_AWS_STORAGE_BUCKET_NAME=YOUR_BUCKET_NAME
DJANGO_AWS_S3_ENDPOINT_URL=s3.amazonaws.com
DJANGO_AWS_LOCATION=A_FOLDER_IN_YOUR_BUCKET
```

If you are using Digital Ocean, you can use their [Spaces](https://www.digitalocean.com/products/spaces/) service in the same way.  From the Digital Ocean dashboard, select the Create - Spaces option and follow the instructions.  Use the default _Restrict File Listing_ option.  Once your Space is created, select the _API_ option and create a Spaces access key.  Copy the Key and Secret values, then fill in the `AWS_*` settings in the `.envs/.production/.django` file as follows:    

```
# .django
# AWS
# ------------------------------------------------------------------------------
DJANGO_AWS_ACCESS_KEY_ID=ASLKFJASLK0WEJ,
DJANGO_AWS_SECRET_ACCESS_KEY=lkj09LSKJDFOSIFLKalkjf0-9SLKJJFOEI
DJANGO_AWS_STORAGE_BUCKET_NAME=YOUR_BUCKET_NAME
DJANGO_AWS_S3_ENDPOINT_URL=nyc3.digitaloceanspaces.com
DJANGO_AWS_LOCATION=A_FOLDER_IN_YOUR_BUCKET
```

Note the `DJANGO_AWS_S3_ENDPOINT_URL` points to the NYC3 location.  If you chose one of the other sites, such as Amsterdam (AMS1) or Singapore (SGP1), use that instead.    

If you choose not to use a cloud storage service, you can fall back to the default Django media storage but you will need to ensure you have enough disk space to handle the media files being served and uploaded.  To do it this way, edit `config/settings/production.py` file and commenting out the following two lines:    

```
# config/settings/production.py
# ...
# MEDIA
# COMMENT OUT THE NEXT TWO LINES TO USE LOCAL MEDIA STORAGE
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
MEDIA_URL = f'%s/%s/%s/' % (AWS_S3_ENDPOINT_URL, AWS_STORAGE_BUCKET_NAME, AWS_LOCATION)
```

> Note that if you choose to store your media files on the server you should probably set up a Docker volume so the data are not held in the Django container.  

## Caddy Web Server    

Finally, have a look at `compose/production/caddy/Caddyfile`.  The default file should look something like this:    

```
www.{$DOMAIN_NAME} {
    redir https://trailstewards.com
}

{$DOMAIN_NAME} {
    proxy / django:5000 {
        header_upstream Host {host}
        header_upstream X-Real-IP {remote}
        header_upstream X-Forwarded-Proto {scheme}
    }
    log stdout
    errors stdout
    gzip
}
```

The [Caddy webserver](https://caddyserver.com/) will automatically obtain an SSL certificate from [LetsEncrypt!](https://letsencrypt.org/) and redirect all requests to a secure HTTPS port on your server.  The configuration file shown above assumes that the server will receive any requests sent to www.yourdomain.name.   If your DNS entries are set this way, everything should work fine.  Note that the `DOMAIN_NAME` environment variable can be found in the `.envs/.production/.caddy` file.    
You can override these settings for whatever configuration you want.  For example, my site is registered to several domains, and I want to serve a static site at www.trailstewards.com and the django site at geo.trailstewards.com.  Here is my custom Caddyfile:    

```
# Caddyfile
www.trailstewards.com, trailstewards.ca, www.trailstewards.ca  {
    redir https://trailstewards.com
}

trailstewards.com {
gzip
root /var/www/html
}

geo.trailstewards.com {
    proxy / django:5000 {
        header_upstream Host {host}
        header_upstream X-Real-IP {remote}
        header_upstream X-Forwarded-Proto {scheme}
    }
    log stdout
    errors stdout
    gzip
}
```

# Step 4 - Build The Containers And Load Some Data    

We should now be able to bring up our server, create a superuser, and load some data.  These steps are covered in the previous post as well.     

```
$ cd geopap_testserver
$ docker-compose -f production.yml build
... lots of docker messages ...
$ docker-compose -f production.yml up -d
$ docker-compose -f production.yml ps
               Name                              Command               State                         Ports                        
---------------------------------------------------------------------------------------------------------------------------------
trailstewardscloudserver_caddy_1      /usr/bin/caddy --conf /etc ...   Up      2015/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp 
trailstewardscloudserver_django_1     /entrypoint.sh /gunicorn.sh      Up                                                         
trailstewardscloudserver_postgres_1   /bin/sh -c /docker-entrypo ...   Up      5432/tcp                                           
trailstewardscloudserver_redis_1      docker-entrypoint.sh redis ...   Up      6379/tcp   
```

The results from the `ps` command shows that we have four containers running.  The caddy webserver is the only one exposed to the internet, while the other containers all communicate on a private docker network.   A [redis](https://redis.io/) container is provided for cache storage of frequently repeated requests.  Now create a superuser.

```
$ docker-compose -f production.yml run --rm django python manage.py createsuperuser
PostgreSQL is up - continuing...
Username: dave
Email address: dave@trailstewards.com
Password: 
Password (again): 
Superuser created successfully.
```

## Load Some Data    

Following the same procedure detailed in the previous post.  Download and unzip [this sample dataset](https://drive.google.com/file/d/1r5EJa4iyERstP3Z_Gd3_UYbHviP92ePX/view?usp=sharing) to your local computer.  Be sure that  [Httpie](https://httpie.org/) is installed, then edit the `load_local.sh` script and replace `user:password` with the superuser name and password you created earlier.  Also replace `http://127.0.0.1:8000` with `https://your.domain.name`.    

```
$ cd data/demo
$ chmod +x load_local.sh
$ ./load_local.sh
```

Follow the process given in the previous post for using the Django admin interface to set up the Profile and ProfileSets, but there is a small twist.  Since the Django admin is a critical security target on a public website, the cookiecutter script creates a random URL for it, rather than /admin.  You can find the `DJANGO_ADMIN_URL` value in the `.envs/.production/.django` file.  If you want to override this _feature_, just edit the value and restart the Django container.    

# What Now?     

This server currently provides minimal functionality, and contributions from other members of the community are welcome, please feel free to fork the repository and send me a pull request.  I'll be adding more postings shortly on how to:    

* enabling social media signups    
* add functionality to the Django code, testing and collaboration    

