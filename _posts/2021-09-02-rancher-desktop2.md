---
layout: post
title: Using Rancher Desktop in an Enterprise environment - Part 2
cover-img: /assets/img/rancher-desktop.png
thumbnail-img: /assets/img/rancher-desktop.png
share-img: /assets/img/rancher-desktop.png
tags: [docker,k3s,K8S, rancher]
---

Yesterday I got Rancher Desktop running, solved an unbuildable Dockerfile issue with some help on Slack and dove into a Certificate issue. Today I'll be continuing with figuring out how to pull from a repository with a self signed CA certificate and I'll try to get an operator running.


This is part 2 of a series where I am trying out Rancher Desktop as a replacement for Docker Desktop.
 - Part 1 can be found here: https://yaron.github.io/2021-09-01-rancher-desktop/

## Encoding
Turns out my hunch from yesterday was wrong. A restart didn't solve the certificate issue, so we're starting with that today. I do another `ps aux|grep lima` to get the SSH command to login to the Alpine instance (the K3S host). I check the contents of `/usr/local/share/ca-certificates/ca.crt` where our cert should end up and notice that it's base64 encoded. That's not what I expected... I check my resources and notice that I am using a Configmap for storing the CA cert. In contrast to a secret, a configmap's data element is not supposed to contain base64 content. There is a binaryData field that should support base64 encoded content, but even though it [documented](https://kubernetes.io/docs/concepts/configuration/configmap/) it doesn't seem to work the same as the normal data field as I can't get it to load the data into an ENV variable in my container. I decide to take the easy road and just use a secret instead of a configmap.
This is what I end up with:
~~~
apiVersion: v1
kind: Secret
metadata:
  name: trusted-ca
data:
  ca.crt: <base64 encoded Ca cert>
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: setup-script
data:
  setup.sh: |
    echo "$TRUSTED_CERT" > /usr/local/share/ca-certificates/ca.crt && update-ca-certificates
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-custom-setup
  labels:
    k8s-app: node-custom-setup
spec:
  selector:
    matchLabels:
      k8s-app: node-custom-setup
  template:
    metadata:
      labels:
        k8s-app: node-custom-setup
    spec:
      hostPID: true
      hostNetwork: true
      initContainers:
      - name: init-node
        command: ["nsenter"]
        args: ["--mount=/proc/1/ns/mnt", "--", "sh", "-c", "$(SETUP_SCRIPT)"]
        image: debian
        env:
        - name: TRUSTED_CERT
          valueFrom:
            secretKeyRef:
              name: trusted-ca
              key: ca.crt
        - name: SETUP_SCRIPT
          valueFrom:
            configMapKeyRef:
              name: setup-script
              key: setup.sh
        securityContext:
          privileged: true
      containers:
      - name: wait
        image: k8s.gcr.io/pause:3.1
~~~
I need to restart K3S now, the easiest way of doing that is just quiting Rancher Desktop (right click on the MacOS top menu bar item) and starting it back up.
After running this I can see the certificate changing to a proper crt file in the Alpine shell I still have open. I delete the pod that was unable to pull, it gets recreated by the deployment and this time it starts up!

## Operations
Next up is the operator. I am running the [Percona for MongoDB Operator](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html) which actually runs without any problems. I can deploy a mongodb cluster from a custom helm chart, but hit some minor issues that I could have expected:
 - I need to change the storage class to "local-path" because there is no "default"
 - I need to set antiAffinity to "none" because there is just one host
 - I need to update the amount of available CPU for the cluster.

This last item can be done through the Docker Desktop GUI, but does reset the cluster which does never work the first time for me and needs to be retried a second time. This also states that we will lose all configuration and workloads, so good thing we have everything stored in code! Not only do you lose everything in the local filesystem, also all deployments, secrets etc. are lost. After applying everything again and restarting Rancher Desktop (fr the CA cert) I can deploy my whole mongodb cluster and everything seems to be running fine!

That's it for today. I don't have any other things I wanted to test, but if there is something I missed let me know! You can contact me on twitter [@yarontal](https://twitter.com/yarontal).
