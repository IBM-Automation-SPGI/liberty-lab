
# Introduction to Labs

Follow this guide to complete the Hands-on Labs for Liberty

In order to complete all labs, different environmnets have been provisioned. Each environemnt is independent from on another, so depending on the skills and interested but we suggest the following order:

## 1. Liberty Introduction

Discover the basics of WebSphere Liberty and how to start Develop on it.

**Liberty Workshop environment - VMWare**

VNC URL: provided by instructor

**Labs:**
- [101-Liberty-Getting-Started](101-Liberty-Getting-Started/README.md)
- [102-Discovery-Liberty](102-Discovery-Liberty/README.md)
- [103-Liberty-devmode](103-Liberty-devmode-VSCode/README.md)

## 2. Liberty Collectives

Discover the Collective architecture for Liberty and how to provide dynamic load balancing and high availability. Finally, experience the zero-migration architecture.

**Enterprise-Deployment-Liberty-Environment**

VNC URL: provided by instructor

**Labs:**
- [111-Liberty-Collectives](111-Liberty-Collectives/README.md)
- [112-Liberty-Dynamic-Routing](112-Liberty-Dynamic-Routing/README.md)
- [113-Liberty-Zero-Migration](113-Liberty-Zero-Migration/README.md)

## 3. Liberty on Containers

Discover how to move Liberty from virtual machine to a container based enviroment.

This labs will introduce you first to Containers and then to OpenShfit, to finally guide you on how to deploy Liberty on and OpenShift cluster.

**131-Intro-Containers** and **132-Intro-OpenShift** are optional if you are already familiar using podman cli to manage containers and using OpenShift

**133-Liberty-on-OpenShift**
Liberty Container Deployment with CP4Apps on OpenShift

VNC URL: provided by instructor

**Labs:**
- [131-Intro-Containers](131-Intro-Containers/README.md)
- [132-Intro-OpenShift](132-Intro-OpenShift/README.md)
- [133-Liberty-on-OpenShift](133-Liberty-on-OpenShift/README.md)

## 4. Liberty InstantOn

Discover InstantOn capability to enable your Liberty application for serveless architectures and event-driven use cases.

This lab will rely on two different environments. The one from the previous Lab and a shared OpenShift Cluster due to version dependencies

**Liberty Container Deployment with CP4Apps on OpenShift**

VNC URL: provided by instructor

**Shared OpenShift Cluster:**
https://console-openshift-console.apps.67a3346aadf42b5f8ef28cdc.eu1.techzone.ibm.com/

username: user[x]
password: passw0rd

**Labs:**
- [141-Liberty-InstantOn-Serverless](141-Liberty-InstantOn-Serverless/README.md)
