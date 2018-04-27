---
layout: post
title:  "Configuring Jenkins with GitHub, Gradle, and Nginx"
date:   2018-04-27 1:00:00
categories: 
tags: featured
image: /assets/images/jenkpic.png
---

I've been wanting to set up a build server for future projects for some time, and decided recently to bite the bullet and try it out, now that my first Minecraft project is at an appropriate point in development. This can serve as a guide for anyone wanting to setup a ForgeGradle project in Jenkins, but most of the content will apply to a broad set of circumstances. 

## Part 1: Jenkins Install
---
I have access to a VPS that I thought was appropriate for the task, but as I would come to find out, it had some particular features that were going to become a problem, specifically that it was running a dated Ubuntu 14 LTS. This meant that there was no native support for Java 8, which is a requirement for Jenkins. 

This meant that I needed to find a way to install Java 8 on Ubuntu 14, which boiled down to adding the WebUpd8 team repo and downloading an install script for Java 8:

{% highlight bash %}
sudo apt-add-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
{% endhighlight %}

Once that was squared away, I moved to getting Jenkins installed on the system. Originally I was going to run from docker, but docker required a version of the Linux kernel that would require me to edit grub records to boot from, and I didn't want to down the entire VPS. Instead, I installed it bare metal and determined that I would have a reverse proxy in front of it. To install, I ran: 

{% highlight bash %}
sudo wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add –
sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install Jenkins
{% endhighlight %}

This was enough to get Jenkins running on the default port of 8080. Now we move to the setup of the reverse proxy through Nginx

## Part 2: Nginx
---

Nginx was arguably the most frustrating part of this setup process, and exposed some of the shortcomings of the dpkg system that I was not aware of, as I don't primarily use Ubuntu/Debian. Originally I just installed Nginx through the official Ubuntu repos, but after trying to get my config file to work, I realized that the version they offer is very very far out of date. What followed was a series of `sudo apt-get remove --purge` and `sudo dpkg -S` trying to get the old version to let go of the system and allow for the install of the latest stable from nginx's repo. Additionally, I had deleted the .conf file, assuming that removing and reinstalling the package would regenerate it. It did not, and indeed does not regenerate any owned config dirs or files of a package. This is good to remember. Eventually I ran `sudo find / | grep nginx | sudo xargs rm -rf
` out of frustration and this finally allowed me to cleanly install nginx from their own repo. 

{% highlight bash %}
sudo add-apt-repository ppa:nginx/stable
sudo apt-get update
sudo apt-get install nginx
{% endhighlight %}

Then we moved to getting Nginx actually configured. 

First, I wanted certs for my proxy, as no one should be running HTTP in this day and age. Let's Encrypt has certbot, a tool I've been meaning to learn for some time. Eventually I came to the conclusion that it was much simpler to run certbot in `--standalone` than the nginx mode, as it didn't mess with any config files for nginx. I used `sudo certbot --standalone certonly`, making sure to `sudo service nginx stop` beforehand, so certbot could use port 80. This installed my certs in `/etc/letsencrypt/live/*domain*/` and allowed me to point to them in my nginx config file. Make sure to specify the subdomain you want to use during the certbot setup. Additionally, you can get a wildcard cert, but I don't plan on running enough services to warrant one. 

Next, we are planning on setting this up on a subdomain, so I added a subdomain A record to my domain, how to do this will depend on your registrar.

We can remove any sites in `/etc/nginx/sites-enabled` since we will be writing our own .conf file for this. In `/etc/nginx/conf.d/jenkins.conf`, I ended up with this file, which will redirect HTTP requests to something a bit more secure:

{% highlight nginx %}

server {
  listen 80;
  server_name your.domain.here;
  return 301 https://$host$request_uri;
}

server {
  listen 443;
  server_name your.domain.here;

  ssl_certificate /etc/letsencrypt/live/ci.thundr.pw/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/ci.thundr.pw/privkey.pem;

  ssl on;
  ssl_session_cache  builtin:1000  shared:SSL:10m;
  ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
  ssl_prefer_server_ciphers on;

  location / {
    proxy_set_header        Host $host:$server_port;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto $scheme;
    proxy_redirect http://localhost:8080 https://your.domain.here;
    proxy_pass              http://localhost:8080;
    # Required for new HTTP-based CLI
    proxy_http_version 1.1;
    proxy_request_buffering off;
    proxy_buffering off; # Required for HTTP-based CLI to work over SSL
    # workaround for https://issues.jenkins-ci.org/browse/JENKINS-45651
    add_header 'X-SSH-Endpoint' 'jenkins.domain.tld:50022' always;
  }
}

{% endhighlight %}

This allows Jenkins to be served over HTTPS! Now, for configuring Jenkins. 

## Part 3: Jenkins Setup
---

This will contain specifics for ForgeGradle.

This section was a bit more straightforward , but still took a decent amount of trial, error, and troubleshooting. First, we can setup all the basics of the project after creating a new project with the default settings. Go to configure, and set the name and the GitHub repo in the SCM section. This is all very straightforward, so I will skip to the unclear parts. 


We need to setup build webhooks for Jenkins so it can build on GitHub push. To do that, we can use the GitHub plugin that comes with Jenkins. First, a checkbox:

![GitHub Build Trigger](/assets/images/1.png){:class="img-responsive"}

Now, we can go on GitHub and setup a webhook for our repo under Settings pointing to `our.domain/github_webhook` that will send a POST whenever we push new code to master. With this set up, we need to specify the gradle commands to run. 

Under the Build section of the configuration for your job, go to `Add build step/Invoke gradle script`:

![Gradle Arguments](/assets/images/2.png){:class="img-responsive"}

I also added another build step that comes before invoke gradle script that runs `rm -rm build/libs` as an Execute shell. This just cleans out the libs folder so jars don't pile up every build. 

Additionally, you're going to want to tell Jenkins where your jars are going to come out under `Post-Build actions/Archive the artifacts`, so it can host them as build artifacts:

![Gradle Arguments](/assets/images/4.png){:class="img-responsive"}

This is it! At this point, the build was working for me, and my artifacts were even being versioned off build number, with this in my build.gradle:

{% highlight java %}

def build_number = 'dev'
if (System.getenv('BUILD_NUMBER') != null)
  build_number = System.getenv('BUILD_NUMBER')
//version= 1.12-1.0.0-143-universal.jar
//or in dev= 1.12-1.0.0-dev-universal.jar
version = "${project.config.mc_version}-${project.config.mod_version}-${build_number}"

{% endhighlight %}

## Conclusion
---

This was a fun project that got my toes wet in build automation and reverse proxying. I probably missed a lot of incredibly crucial details, but I won't know until something breaks. 
