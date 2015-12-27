---
layout: post
cover: 'assets/images/cover7.jpg'
title: Running Jenkins Locally with Docker
date:   2015-12-26 10:18:00
tags: jenkins
subclass: 'post tag-test tag-content'
categories: 'casper'
navigation: True
logo: 'assets/images/ghost.png'
---

#Why even do this?
Jenkins is great but sometimes you need to test things in a throw away environment so you don't blow up your whole pipeline. Plugin and Jenkins updates happen often, and once you have been burned by a few of them, you start seeing the value of being able to quickly test new versions in an isolated environment.

##Step 1
Make sure you have a docker-machine where you can run the Jenkins master container. For this tutorial we will have a docker-machine VM for the Jenkins master and a seperate one for the Jenkins slave container.

The [offical jenkins container](https://hub.docker.com/_/jenkins/) has good instructions on how you can persist your jenkins data between creating and destroying containers and upgrading. That is outside the scope of this tutorial. We are just going to get this thing up and running.

```
docker-machine create --driver virtualbox dev
```
Creates a docker-machine vm called `dev` to use for the Jenkins master

```
$(docker-machine env dev)
```
This will setup your current docker-client to connect to the `dev` docker machine.

```
docker run -d --name jenkins-docker -p 8080:8080  jenkins
```
This will start the latest jenkins container and bind it to port 8080 on your docker-machine ip. To get the ip of your `dev` docker machine you can execute the following command
```
docker-machine ip dev
```

My ip currently is 192.168.99.100 so when I visit http://192.168.99.100:8080 I will see the following.

{<1>}![jenkins_startup_image](/content/images/2015/11/Screen-Shot-2015-11-17-at-11-45-48-AM.png)

##Step 2
Now that you have a  working master we need a docker-machine to host the docker slaves that Jenkins will spin up. I want to keep this seperate because if you are running DIND(Docker in Docker) you could potentially affect your Jenkins master docker container if your not careful. So I am extra cautious and spin up another VM for my Jenkins docker slaves.

```
docker-machine create -d virtualbox \
    --engine-opt host=tcp://0.0.0.0:4243 \
    --engine-env DOCKER_TLS=no \
    jenkins-slaves
```

So to explain more whats going on here. We can set docker-engine options with the `engine-opt` command. Specifically here we are telling docker to listen on a port that we know about `4243`. Jenkins will interact with the docker deamon through this port using [docker remote api](https://docs.docker.com/engine/reference/api/docker_remote_api/).

The next thing we are doing is passing an environment variable to the docker-machine VM. `DOCKER_TLS=no` turns off TLS for the docker-daemon and allows us to interact with the api without encryption. Although the Jenkins plugin does have a 'credentials' configuration, I have not been able to figure out how to connect Jenkins to docker with TLS enabled... yet another reason to have a seperate VM just for Jenkins to control.

##Step 3
Now we actually get to configure Jenkins to connect and spin up slaves!

Install the Jenkins docker plugin.

**Jenkins Home -> Manage Jenkins -> Manage Plugins ->**

Or a direct url of http://192.168.99.100:8080/pluginManager/available

Remember that is my local ip not yours. You need to make sure you connecting to our dev docker-machine ip.


Search for docker in the search box and then check the `Docker Plugin` as shown below.

{<2>}![Docker_plugin](/content/images/2015/11/Screen-Shot-2015-11-17-at-12-03-06-PM.png)

Then click `Install without restart` button and the plugin will be installed. 

##Step 4
Docker plugin is installed, now we need to make sure it connects and launches a container on our `jenkins-slave` docker-machine VM.

**Jenkins Home -> Manage Jenkins -> Configure System**

Or a direct url of
http://192.168.99.100:8080/configure

Scroll to the very bottom of the page and find an option labeled `Add a new cloud`
{<3>}![add_a_new_cloud](/content/images/2015/11/Screen-Shot-2015-11-17-at-12-14-33-PM.png)


Select `Docker` and now you will see a new set of configuration options that we will need to fill in.

**Name:** jenkins-slave

**Docker URL**: http://`ipAddress of jenkins-slave docker-machine VM`:4243

That is all you need to now hit the `Test Connection` button. My configuration with the result of pressing the `Test Connection` button is shown below.

{<4>}![connection_tested](/content/images/2015/11/Screen-Shot-2015-11-17-at-12-23-01-PM.png)

Now Jenkins can connect to its salve docker daemon! We just have to tell jenkins which container to bring up and use as a slave!

##Step 5

We will now configure the Jenkins DIND (Docker in Docker) image so that we can run docker commands in our jenkins builds. The image we will be working with can be found [here](https://hub.docker.com/r/tehranian/dind-jenkins-slave/)

On the same Jenkins configuration page press the `Add Docker Template` button.

**Docker Image:** tehranian/dind-jenkins-slave
**label:** docker

Now you need to click the `Container Settings` button. You will now see a bunch of new configuration options.

The only one we are interested in is the `Run container privileged` checkbox.

{<5>}![run_privliged](/content/images/2015/11/Screen-Shot-2015-11-17-at-12-54-59-PM.png)

Now we need to configure 'Credentials'. Hit the `Add` button next to the 'Credentials' config.
