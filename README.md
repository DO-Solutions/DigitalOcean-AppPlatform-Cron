# DigitalOcean-AppPlatform-Cron-Worker
<!-- <div id="top"></div> -->
<!--
*** Thanks for checking out the Best-README-Template. If you have a suggestion
*** that would make this better, please fork the repo and create a pull request
*** or simply open an issue with the tag "enhancement".
*** Don't forget to give the project a star!
*** Thanks again! Now go create something AMAZING! :D
-->


<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://digitalocean.com/">
    <img src="./assets/DO_Logo-Blue.png" alt="Logo" >
  </a>

<h3 align="center">DigitalOcean | App Platform Cron Worker</h3>

  <p align="center">
    App Platform allows you to Build, deploy, and scale apps quickly using a simple, fully-managed infrastructure solution.
    <br>This tutorial shows you how to <b>run scheduled jobs on App Platform using a Cron Worker.</b>
    <br />
    <a href="https://docs.digitalocean.com/tutorials/app-platform/"><strong>Explore more App Platform tutorials»</strong></a>
    <br />
    <a href="https://www.digitalocean.com/product-tours/app-platform"><strong>Quick App Platform tour»</strong></a>
  
  </p>
</div>

# Getting Started


## Architecture diagram
![architecture](./assets/cron-architecture.png)

## Introduction

In this tutorial, we'll guide you through the process of setting up a job scheduler in App Platform using a docker container that runs cron as an App Platform [Worker](https://docs.digitalocean.com/products/app-platform/how-to/manage-workers/). We'll talk you through the steps required to build the docker container in case you want to modify it and deploy the scheduler as an App Platform Worker using the smallest/cheapest container size. Defining your own scheduled jobs is as easy as modifying the included `crontab` file.


## Prerequisites

1. A DigitalOcean account ([Log in](https://cloud.digitalocean.com/login))
2. doctl CLI([tutorial](https://docs.digitalocean.com/reference/doctl/how-to/install/))

# Quick / Easy version

1. Fork this [docker-cron](https://github.com/DO-Solutions/docker-cron) repo
2. Add the following to your [App Spec](https://docs.digitalocean.com/products/app-platform/reference/app-spec/) (yaml):

```yaml
workers:
- dockerfile_path: Dockerfile
  github:
    branch: main
    deploy_on_push: true
    repo: <your-github-username>/docker-cron
  instance_count: 1
  instance_size_slug: basic-xxs
  name: docker-cron
  source_dir: /
```

3. Modify `crontab` in the forked repo to add your own cron jobs.

# Hard / detailed version

## Fork docker-cron

First import or fork [https://github.com/DO-Solutions/docker-cron](https://github.com/DO-Solutions/docker-cron) to a new Git repo on Github or Gitlab so that we can deploy our Docker Cron Worker

## Modify `Dockerfile`
The provided Dockerfile is used by App Platform to build and run our Cron Worker, it should be modified to your liking

* `ubuntu:22.04` base image is pulled
* `cron` and `curl` is installed, you can modify this line to include any other tools you may need to run your scheduled jobs

```Dockerfile
FROM ubuntu

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get -y --no-install-recommends install -y cron curl \
    # Remove package lists for smaller image sizes
    && rm -rf /var/lib/apt/lists/* \
    && which cron \
    && rm -rf /etc/cron.*/*

COPY crontab /hello-cron
COPY entrypoint.sh /entrypoint.sh

RUN crontab hello-cron
RUN chmod +x entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]

# https://manpages.ubuntu.com/manpages/trusty/man8/cron.8.html
# -f | Stay in foreground mode, don't daemonize.
# -L loglevel | Tell  cron  what to log about jobs (errors are logged regardless of this value) as the sum of the following values:
CMD ["cron","-f", "-L", "2"]
```

## Modify `crontab` to define your own cron jobs

```sh
* * * * * curl http://sample-nodejs:8080 >/proc/1/fd/1 2>/proc/1/fd/2
# An empty line is required at the end of this file for a valid cron file.

```


# Deploy to App Platform
## Modify your App Spec
We're going to assume that you're adding docker-cron to an existing App Platform app. Use `doctl` to retrieve your existing apps App Spec and add docker-cron.

1. Retrieve App ID

`doctl apps list`

2. Use that ID to retrieve your apps App Spec

`doctl apps spec get b6af73dc-8aba-4237-8dc9-b632ad379bd5 > appspec.yaml`

```yaml
alerts:
- rule: DEPLOYMENT_FAILED
- rule: DOMAIN_FAILED
name: walrus-app
region: nyc
services:
- environment_slug: node-js
  git:
    branch: main
    repo_clone_url: https://github.com/digitalocean/sample-nodejs.git
  http_port: 8080
  instance_count: 1
  instance_size_slug: basic-xxs
  name: sample-nodejs
  routes:
  - path: /
  run_command: yarn start
  source_dir: /
  ```
  
3. Add the Docker-cron worker

```yaml
alerts:
- rule: DEPLOYMENT_FAILED
- rule: DOMAIN_FAILED
name: walrus-app
region: nyc
services:
- environment_slug: node-js
  git:
    branch: main
    repo_clone_url: https://github.com/digitalocean/sample-nodejs.git
  http_port: 8080
  instance_count: 1
  instance_size_slug: basic-xxs
  name: sample-nodejs
  routes:
  - path: /
  run_command: yarn start
  source_dir: /
workers:
- dockerfile_path: Dockerfile
  github:
    branch: main
    deploy_on_push: true
    repo: <your-github-username>/docker-cron
  instance_count: 1
  instance_size_slug: basic-xxs
  name: docker-cron
  source_dir: /
  ```
  
4. Update your app to deploy Docker-cron Worker
 
`doctl apps update b6af73dc-8aba-4237-8dc9-b632ad379bd5 --spec appspec.yaml`

# Verify worker functionality
We can use `doctl` to retrieve our runtime logs and verify our cron is running, by default it will output to console

`doctl apps logs b6af73dc-8aba-4237-8dc9-b632ad379bd5 --type=run`


<p align="right">(<a href="#top">back to top</a>)</p>



<!-- USAGE EXAMPLES -->



<!-- CONTACT -->
# Contact

Jack Pearce, Solutions Engineer - jpearce@digitalocean.com

<p align="right">(<a href="#top">back to top</a>)</p>
