---
layout: post
title: Using Rancher Desktop in an Enterprise environment
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [docker,k3s,K8S, rancher]
---

So this is probably going to be part of a series, as this is an alpha product which I am trying to use in a hostile environment. I have no idea what I am going to run into, so we'll see how long I can keep this up.

##Why?
Good question! I was using docker desktop and was looking into alternatives that work on MacOS. Rancher Desktop looks promising and like something that could also be used by less technical users.

##Invalid user
I set up Rancher Desktop, connect to it with Lens and all looks good. Now let's build a docker image. Rancher Desktop comes bundled with KIM, which allows images to be available in the k3s cluster directly, so that sounds good. I run Kim build and all looks good. I run Kim images,..... no image... Oh let's try Kim images -a. Also not our image!?
I am sure I am doing something wrong, so I start looking for logs. Kim is running in the cluster, so we can just do a kubectl log. After some digging I give up, connect to the rancher slack and join the Kim channel. I post a writeup of my problem and decide to test some other Dockerfile as well. This one works!?
So now might be a good time to show the Dockerfile I was building.


We need to change the uid to a numeric uid because we have an OPA (Open Policy Agent) rule that forbids us from using uid 0 or a non numeric one. Turns out that we found our first bug!
https://github.com/rancher/kim/issues/74
The help in slack and the thorough writeup on GitHub where a good first impression though!

##No authority
After having build our image, we're on to running one! I get a yaml file of a deployment that I have laying around and run kubectl apply on it. I open up Lens to see if it starts, but I see the dreaded yellow warning ⚠️. X509 error when pulling the image...
I was trying to use an image from a local repository with a certificate signed by an internal Certificate Authority (CA). This CA is obviously not trusted yet on the Alpine Linux that is ran by Lima and on which K3S is running. I looked through the documentation of Lima to see if I can hook in there somewhere, but the only way I could even login to the Alpine VM/container was by doing a ps aux|grep "Lima" and copy pasting the ssh command that was in there.
After that dead end I tried to go the K8S way of using a daemon set that mounts the host's CA cert directory, copies the CA cert to it and runs update-ca-certificates. I could see the new cert in the Alpine instance, but the pull still didn't work. There where some errors in the CA update container's logs though, so let's look at those.
~~~
WARNING: ca-certificates.crt does not contain exactly one certificate or CRL: skipping
~~~
I Google on the error and find a GitHub issue!
https://github.com/gliderlabs/docker-alpine/issues/30
Reading through the issue though, I figure out that it's a warning that's expected behavior. The file in the error is supposed to contain all CA certs, and is supposed to be skipped when creating the new bundle, so that was a waste of time...

It's time for dinner so I'm calling it a day. Seems like I was right and this is going to be a series. I'll  get back to this tomorrow and see if a good night's sleep will help. Now that I think of it, that might actually be the solution. Restarting the cluster might have K3S read the CA bundle and trust our custom CA cert. We'll see!
