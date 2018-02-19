<!--
[![Build Status](https://travis-ci.org/IBM/watson-banking-chatbot.svg?branch=master)](https://travis-ci.org/IBM/watson-banking-chatbot)
![IBM Cloud Deployments](https://metrics-tracker.mybluemix.net/stats/527357940ca5e1027fbf945add3b15c4/badge.svg)
-->
# Building Cloud Native Applications with Kubernetes

No application is an island. In this journey, we'll explore how to deploy a system of services to kubernetes, including what happens next: How to add resilience, monitoring, end-to-end trace, content-aware routing, etc.

We're basing this journey on [Game On! A throwback text adventure](https://gameontext.org) built from the ground up to facilitate exploration of Cloud Native systems and concepts. The application will be deployed to a Kubernetes cluster, and operates in several levels: there is a set of replaceable platform-provided services, like backing storage, that vary depending on where the application runs; and there is a set of core game services. Most of the core game services are written with Java 8, and run on [OpenLiberty](https://openliberty.io/) with [MicroProfile](http://microprofile.io/) features. There are a few that are simple nginx processes serving static content from different projects.

There is an additional pseudo-layer for services that extend the game by defining "rooms". Rooms in the game are single-service sandboxes for experimenting with backend technologies. In this journey, you'll also experiment with adding your own room to the system.

Using this Code Pattern, you will: 

* Deploy a system of microservices to Kubernetes using `kubectl`
* Deploy a system of microservices to Kubernetes using helm charts
* Make changes to an existing Kubernetes cluster
* Create and deploy your own service to a Kubernetes cluster

<!--Remember to dump an image in this path-->
![](/images/architecture.png)

### Flow

1. _Game Play:_ The user visits the GameOn! deployment on Kubernetes through a proxy, which surfaces the collection of service APIs as a single facade for the entire application. The proxy retrieves the front-end client, a single-page web application, from a stand-alone nginx process and serves it to the user's browser.
2. _Game Play:_ The user pushes the button to play the game. The user selects a  Social Sign-On service from a list of options. The front-end client makes a RESTful request to the Auth service to begin the login process. 
3. _Game Play:_ The Auth service uses the OAuth2 protocol to authenticate the user with the selected Social Sign-On service. At the end of the login process, the Auth service returns a signed JWT that identifies the user.
4. _Game Play:_ The front-end client includes the signed JWT in REST requests to the Player service to retrieve, and create if necessary, the user's profile.
5. _Game Play:_ (a) The front-end client establishes a WebSocket connection with the Mediator service to begin playing the game. (b) The Mediator establishes WebSocket connections to backend rooms, and passes messages back and forth.
6. _Game Play:_ The Mediator manages connections to individual microservices providing each room. It calls an API on the Map service to find neighboring rooms.
7. _Game Play:_ When a user changes location, the Mediator calls an API on the Player service to update the user's stored location. 
8. _Room development:_ The user creates and deploys a new service to create a new room for the game (from scratch or using an exiting walkthrough). 
9. _Room development:_ The user uses the Room Management panel in the front-end client to make a RESTful request to the Map service to register their service.
10. _Room development:_ The Map service makes a RESTful request to the Player service to check that the user has required permissions. If all is well, the room is registered and added to the game's map. 

<!--Update this section-->
### Included components

* [Apache CouchDB](http://couchdb.apache.org/): An open-source database software that focuses on ease of use and having an architecture that “completely embraces the Web.”
* [Apache Kafka](https://kafka.apache.org/): A distributed streaming platform for building pipelines and apps.
* [Istio](https://istio.io/): An open platform to connect, manage, and secure microservices.
* [JAX-RS](http://cxf.apache.org/docs/jax-rs.html): Java API for RESTful Web Services or JAX-RS is an API specifications for creating web services using the Representational State Transfer (REST) architectural pattern.
* [Kubernetes Cluster](https://console.bluemix.net/docs/containers/container_index.html): Create and manage your own cloud infrastructure and use Kubernetes as your container orchestration engine.
* [MicroProfile](http://microprofile.io/): Optimize Enterprise Java for a microservices architecture.
* [Swagger](https://swagger.io/): A framework of API developer tools for the OpenAPI Specification that enables development across the entire API lifecycle.

<!--Update this section-->
## Featured technologies

* [Cloud](https://www.ibm.com/developerworks/learn/cloud/): Accessing computer and information technology resources through the Internet.
* [Container Orchestration](https://www.ibm.com/cloud-computing/bluemix/containers): Automating the deployment, scaling and management of containerized applications.
* [Containers](https://www.ibm.com/cloud-computing/bluemix/containers): Virtual software objects that include all the elements that an app needs to run.
* [Java](https://java.com/): A secure, object-oriented programming language for creating applications.
* [Messaging](https://developer.ibm.com/messaging/message-hub/): Messaging is a key technology for modern applications using loosely decoupled architecture patterns such as microservices.
* [Microservices](https://www.ibm.com/developerworks/community/blogs/5things/entry/5_things_to_know_about_microservices?lang=en): Collection of fine-grained, loosely coupled services using a lightweight protocol to provide building blocks in modern application composition in the cloud.

<!--Update this section when the video is created
# Watch the Video
[![](http://img.youtube.com/vi/Jxi7U7VOMYg/0.jpg)](https://www.youtube.com/watch?v=Jxi7U7VOMYg)
-->

## Prerequisite

* [Docker](https://docs.docker.com/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* A Kubernetes cluster. Consider the following, or choose your own:
  - [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) for local testing
  - [IBM Cloud Private](https://github.com/IBM/Kubernetes-container-service-GitLab-sample/blob/master/docs/deploy-with-ICP.md) 
  - [IBM Bluemix Container Service](https://github.com/IBM/container-journey-template) 
    
  The code here is regularly tested against [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) using Travis CI.
  
  [Get more help setting up clusters](https://github.com/gameontext/gameon/tree/master/kubernetes#set-up-a-kubernetes-cluster)

## Steps

1. [Clone the gameontext/gameon repository](#1-clone-the-gameontextgameon-repository)
2. [Setup the game to work with your cluster](#2-setup-the-game-to-work-with-your-cluster)
3. [Start the game](#3-start-the-game)
4. [Wait for services to be available](#4-wait-for-services-to-be-available)
5. [Play!](#5-play)
6. [Stop the game](#6-stop-the-game)

### 1. Clone the gameontext/gameon repository

        $ git clone https://github.com/gameontext/gameon.git
        $ cd gameon                  # cd into the project directory
        $ ./go-admin.sh choose 2     # choose Kubernetes
        $ eval $(./go-admin.sh env)  # set aliases for admin scripts
        $ alias go-run               # confirm kubernetes/go-run.sh

Instructions below will reference `go-run`, the alias created above. Feel free to invoke `./kubernetes/go-run.sh` directly if you prefer.

The `go-run.sh` and `k8s-functions` scripts encapsulate setup and deployment of core game services to kubernetes. Please do open the scripts to see what they do! We opted for readability over shell script-fu for that reason.

### 2. Setup the game to work with your cluster

        $ go-run setup

This will ensure you have the right versions of applications we use, prompt to use helm or not, and create a cerficate for signing JWTs.

### 3. Start the game

        $ go-run up

This step will also create a `gameon-system` name space and a generic kubernetes secret containing that certificate.

### 4. Wait for services to be available

        $ go-run wait

### 5. Play!

Visit your external cluster IP address and poke around. Now that the game is working locally, let's venture off into other related adventures:

* [Changing service routing Istio]()
* [Viewing application metrics]()
* [Create a room service]()
  - [Deploy into the existing local cluster]()
  - [Deploy to a public Kubernetes cluster]()


### 6. Stop the game

        $ go-run down

## References
* [Game On! Text adventure](https://gameontext.org)
* [Game On! Application architecture](https://book.gameontext.org/microservices/)
* [How Game On! came to be](https://book.gameontext.org/chronicles/)
* [Creating a new room](https://book.gameontext.org/walkthroughs/createRoom.html)

<!--keep this-->

# License
[Apache 2.0](LICENSE)
