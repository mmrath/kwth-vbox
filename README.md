# Kubernetes The Hard Way on VirtualBox

This repository contains the files to implement Kelsey Hightower's "Kubernetes the Hard Way" [project](https://github.com/kelseyhightower/kubernetes-the-hard-way) on VirtualBox. Kelsey's project implements Kubernetes (K8s) on Google Cloud and an implementation on VirtualBox has to be adapted in a number of ways, which I documented here.

I also reworked the sequence of actions taken in the original repository to be more step-by-step, for example Kelsey generates all necessary certificates in one step. I prefer to generate them on demand when they become necessary.

I used a Ubuntu 18.04 server at DigitalOcean with 16 GB Ram and 6 CPUs as the hosts for the Virtual Machines (VMs) that compromise the K8S cluster.
