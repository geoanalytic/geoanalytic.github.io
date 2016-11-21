---
layout: post
title: A production ready web mapping toolkit - Part 1
---

The long term goal of this series of posts is to develop a set of ready to use tools that will enable the rapid setup of web mapping servers that can be prototyped, developed, and moved to production without a lot of embarassing oversights.  The key technologies that I want to use are:   

* [PostgreSQL](https://www.postgresql.org)/[PostGIS](http://www.postgis.net)   
* [Django](https://www.djangoproject.com/)/[Python](https://www.python.org/)/[GeoDjango](https://docs.djangoproject.com/en/1.10/ref/contrib/gis/)   
* [Docker](https://www.docker.com/)  
* [Leaflet](http://leafletjs.com/) and [Openlayers](https://openlayers.org/)  
* TLS/HTTPS using [LetsEncrypt!](https://letsencrypt.org/)

This first post will deal with getting the basic server set up.  By the end of this post, you should have a cloud based server, running Docker, with separate containers running PostGIS, Django, Nginx, and Certbot.  Later posts will deal with writing Django apps to handle back end tasks and hooking up javascript mapping libraries to our web pages.   

----------   
# Get Yourself A Droplet From The Digital Ocean  
I'm going to assume that all the work will be done in the cloud.  If you already have a cloud service you are using, then you may have to adapt some of the commands to work with your service.  If you don't have a cloud service account, may I suggest [DigitalOcean](https://www.digitalocean.com/)?  The service is easy to use and low cost and, best of all, if you sign up through [my referral link](https://m.do.co/c/07e7a94179df) you will recieve a $10 credit to get started.  Since this tutorial will run on the lowest cost server on DO, that will cover you for two months of free experimentation.   

So, you have your Digital Ocean account?  Great, now;  
1. click on the __Create Droplet__ link   
2. click the __One-click Apps__ tab   
3. select __Docker 1.12.3 on 16.04__ (note these version numbers will change in the future)   
4. select the $5 __size__ option - this gives you one CPU with 512MB of RAM and a 20GB SSD  
5. select the __datacenter region__ nearest to you  
6. (optional) upload an __SSH key__ for secure login   
7. choose a __host name__, such as docker-demo   
8. click the __create__ button  




![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.