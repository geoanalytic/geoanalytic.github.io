---
layout: post
comments: true
title: A reference server for geopaparazzi cloud profiles
---

We have been working with the [geopaparazzi mobile app](https://github.com/geopaparazzi/geopaparazzi).  This is a very interesting app with a lot of capabilities, but setting it up is a pretty daunting prospect for a casual user.  The first order of business is to make the download/upload of maps and user data easier.  This is done using 'cloud profiles' which collect all the basemap, spatialite DBs, forms and other requirements into a listing served as a JSON stream that references the download links.  This approach can be satisfied with hand assembled files hosted on a cloud service.    

To do anything more complex on the server side will require some infrastructure, including a database and scripting environment.  To make it easier for prospective developers to get a server set up with a minimum of effort without sacrificing security concerns, I have created a [cookiecutter recipe](https://github.com/audreyr/cookiecutter) to quickly stand up a server for testing or production using the best open source code.     

# Features    

* PostgreSQL/PostGIS database    
* Secure web server (TLS/HTTPS) using Caddy and free LetsEncrypt! certificates    
* Django/Python web framework for powerful scripting and templating   
* Mobile ready web forms with Crispyforms and Bootstrap    
* Advanced user registration with django-allauth     

# Getting Started   

To try the server out, you will need to install [Docker](https://docs.docker.com/install/) and [Docker-compose](https://docs.docker.com/compose/install/).  Next, install [cookiecutter](https://cookiecutter.readthedocs.io/en/latest/)    

```
$ pip install "cookiecutter>=1.4.0"
```

Now run it against this repo:

```
$ cookiecutter https://github.com/geoanalytic/cookiecutter-geopaparazzi-server
```

You will be asked for some values, here is what I did.  Important things right now:   

* use docker and whitenoise   
* do not use mailhog, sentry, heroku, or travisci   

```
project_name [My Awesome Project]: geopap_testserver
project_slug [geopap_testserver]: 
description [Behold My Awesome Project!]: 
author_name [Daniel Roy Greenfeld]: David Currie
email [david-currie@example.com]: dave@trailstewards.com
domain_name [example.com]: trailstewards.com
version [0.1.0]: 
Select open_source_license:
1 - MIT
2 - BSD
3 - GPLv3
4 - Apache Software License 2.0
5 - Not open source
Choose from 1, 2, 3, 4, 5 [1]: 
timezone [UTC]: 
windows [n]: 
use_pycharm [n]: 
use_docker [n]: y
postgresql_version [10.3]: 
Select js_task_runner:
1 - None
2 - Gulp
3 - Grunt
Choose from 1, 2, 3 [1]: 
custom_bootstrap_compilation [n]: 
use_compressor [n]: 
use_celery [n]: 
use_mailhog [n]: 
use_sentry_for_error_reporting [y]: n
use_whitenoise [y]: 
use_heroku [n]: 
use_travisci [n]: 
keep_local_envs_in_vcs [y]: 
 [SUCCESS]: Project initialized, keep up the good work!
```

Now cd into the directory that was just created for you using the project_slug name, and build a local server

```
$ cd geopap_testserver
$ docker-compose -f local.yml build
```

This will take a while and will download a bunch of docker stuff.  When the build is finished, fire up the local service and run the tests.    

```
$ docker-compose -f local.yml up -d
Creating network "geopaptestserver_default" with the default driver
Creating volume "geopaptestserver_postgres_data_local" with default driver
Creating volume "geopaptestserver_postgres_backup_local" with default driver
Creating geopaptestserver_postgres_1
Creating geopaptestserver_django_1
$ docker-compose -f local.yml run django py.test
PostgreSQL is up - continuing...
Test session starts (platform: linux, Python 3.6.1, pytest 3.5.0, pytest-sugar 0.9.1)
Django settings: config.settings.test (from ini file)
rootdir: /app, inifile: pytest.ini
plugins: sugar-0.9.1, django-3.1.2

 geopap_testserver/users/tests/test_admin.py ✓✓                                                                                   4% ▍         
 geopap_testserver/users/tests/test_models.py ✓✓                                                                                  7% ▊         
 geopap_testserver/users/tests/test_urls.py ✓✓✓✓✓✓✓✓                                                                             22% ██▎       
 geopap_testserver/users/tests/test_views.py ✓✓✓                                                                                 28% ██▊       
 profiles/tests/test_api.py ✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓                                                                              70% ███████▏  
 profiles/tests/test_models.py ✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓✓                                                                                 100% ██████████

Results (27.92s):
      54 passed
```

Everything seems to be working?  Now create a superuser.

```
$ docker-compose -f local.yml run django python manage.py createsuperuser
PostgreSQL is up - continuing...
Username: dave
Email address: dave@trailstewards.com
Password: 
Password (again): 
Superuser created successfully.
```

Point your browser to http://127.0.0.1:8000 and you should see the default welcome page.    
You can construct a profile through the admin interface athttp://127.0.0.1:8000/admin, login as the superuser created above.    
The RESTful urls include:    

* http://127.0.0.1:8000/projects/    
* http://127.0.0.1:8000/tags/    
* http://127.0.0.1:8000/otherfiles/    
* http://127.0.0.1:8000/spatialitedbs/    
* http://127.0.0.1:8000/profiles/    
* http://127.0.0.1:8000/myprofiles/    (requires login)




