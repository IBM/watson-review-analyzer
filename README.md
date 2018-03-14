[![Build Status](https://travis-ci.org/IBM/watson-review-analyzer.svg?branch=master)](https://travis-ci.org/IBM/watson-review-analyzer)

# Create a Review Analyzer with Watson Discovery

In this Code Pattern, we will create a Node.js app that takes the reviews from an online shopping website, Amazon, and feeds them into the Watson Discovery service. The reviews will be stored in a Cloudant or couchdb database. The Watson Discovery service will show the overall sentiments of the reviews. The sample application will do all the reading of reviews for you and will give an overall insight about them. The Code Pattern can be useful to developers that are looking into processing multiple documents with Watson Discovery.

When the reader has completed this Code Pattern, they will understand how to:

* Interact with Watson Discovery using Watson's Node SDK
* Build a user interface around the result of Watson Discovery
* Deploy the app in Kubernetes
* Deploy and connect a CouchDB in the same instance of Kubernetes Cluster

<!--Remember to dump an image in this path-->
![Architecture](/docs/app-architecture.png)

## Flow
1. The user deploys the app in Kubernetes. The user interacts with the user interface of the app.
2. The user enters the product ID and the app would start getting the reviews of the Product.
3. The app then stores the reviews in a database for later use.
4. The app starts to upload the reviews in Watson Discovery.
5. After Watson Discovery finishes processing the reviews, the app then stores the result (General Sentiment and Top Entities) in the CouchDB. The user will see the result in the UI.

## Included components
* [Watson Discovery](https://www.ibm.com/watson/services/discovery/):  A cognitive search and content analytics engine for applications to identify patterns, trends, and actionable insights.
* [Kubernetes Cluster](https://console.bluemix.net/containers-kubernetes/catalogCluster): Create and manage your own cloud infrastructure and use Kubernetes as your container orchestration engine.

## Featured technologies
* [Node.js](https://nodejs.org/): An open-source JavaScript run-time environment for executing server-side JavaScript code.
* [Databases](https://en.wikipedia.org/wiki/IBM_Information_Management_System#.22Full_Function.22_databases): Repository for storing and managing collections of data.
* [Cloud](https://www.ibm.com/developerworks/learn/cloud/): Accessing computer and information technology resources through the Internet.

# Prerequisite

Create a Kubernetes cluster with either [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube) for local testing, or with [IBM Bluemix Container Service](https://github.com/IBM/container-journey-template/blob/master/README.md) to deploy in cloud. The code here is regularly tested against [Kubernetes Cluster from Bluemix Container Service](https://console.ng.bluemix.net/docs/containers/cs_ov.html#cs_ov) using Travis.

Install [Docker](https://www.docker.com) by following the instructions [here](https://www.docker.com/community-edition#/download) for your preferred operating system.  
You would also need a Docker Registry. [Docker Hub](https://hub.docker.com/) lets you create one quickly.

# Steps

1. [Clone the repo](#1-clone-the-repo)
2. [Create IBM Cloud services](#2-create-ibm-cloud-services)
3. [Build your Docker image](#3-build-your-docker-image)
4. [Configure deployment files](#4-configure-deployment-files)
5. [Deploy the application](#5-deploy-the-application)
6. [Search for a product in Amazon](#6-search-for-a-product-in-amazon)

## Deploy to IBM Cloud

[![Deploy to Bluemix](https://metrics-tracker.mybluemix.net/stats/7044d7c07bc30cff6a6c2a52e804ad8a/button.svg)](https://bluemix.net/deploy?repository=https://github.com/IBM/watson-review-analyzer.git)

1. Create the following service:
    * [**Watson Discovery**](https://console.ng.bluemix.net/catalog/services/discovery)
    * [**IBM Cloud Kubernetes**](https://console.bluemix.net/containers-kubernetes/catalog/cluster)
      > You can use an existing IBM Cloud Kubernetes cluster

2. Press the above `Deploy to IBM Cloud`
    > The toolchain uses GitHub for its source control. You may be asked to authenticate the toolchain to use your account. The toolchain will clone this repo and will be used for its deployment.
3. Fill out the following fields.
    * If you don't have an API key for your account, create one [here](https://console.ng.bluemix.net/iam/#/apikeys)
    * For the cluster name, choose the one you have created
    * For the image registry namespace, use an existing one if you have previously created an [IBM Cloud Container Registry](https://console.bluemix.net/containers-kubernetes/registry/start) or use a unique name for yourself. The toolchain will create one for you if the namespace is existing
      > Toolchain will fail if the namespace is taken
    * For the Discovery username and Discovery password, use the one you have just created
![fields](docs/fields.png)

4. The IP address of your app can be found in the logs of the delivery pipeline's deploy stage.
![deploy](docs/deploy.png)
![link](docs/link.png)

## Deploy to Kubernetes

### 1. Clone the repo

```
$ git clone https://github.com/IBM/watson-review-analyzer
$ cd watson-review-analyzer/
```

### 2. Create IBM Cloud services

Create the following service:

* [**Watson Discovery**](https://console.ng.bluemix.net/catalog/services/discovery)
* [**IBM Cloud Kubernetes**](https://console.bluemix.net/containers-kubernetes/catalog/cluster)

### 3. Build your Docker image
Login to Docker using your username and password for your Docker Registry.

```
$ docker login
```

The Node.js app will be packaged into a Docker image. This will be used by the Kubernetes Cluster.

```
$ docker build -t YOUR_DOCKERHUB_USERNAME/watson-review-analyzer:1.0 .
$ docker push YOUR_DOCKERHUB_USERNAME/watson-review-analyzer:1.0
```

### 4. Configure deployment files
Setup kubectl to use your cluster

```
$ bx cs cluster-config <your-cluster-name>
```
Use the configuration for your cluster by exporting the environment variables - copy and paste the full line starting with:

```
$ export KUBECONFIG=
```

Add your Watson Discovery credentials in `config.json.sample` and rename it to `config.json`:

![Discovery Credentials](docs/discovery-credentials.png)

`config.json`:
```
{
  "discoveryUsername" : "YOUR_WATSON_DISCOVERY_USERNAME",
  "discoveryPassword" : "YOUR_WATSON_DISCOVERY_PASSWORD"
}
```

Create configmap in Kubernetes for `config.json`

```
$ kubectl create configmap watson-discovery-config --from-file=config.json
```

Modify `watson-review-analyzer.yaml`. In the line where you specify the image name, use the docker image you just built.  
Change `horeaporutiu/watson-review-analyzer:2.1` to `YOUR_DOCKERHUB_USERNAME/watson-review-analyzer:1.0`.

<pre>
    spec:
      containers:
        - image: <b>horeaporutiu/watson-review-analyzer:2.1</b>
          imagePullPolicy: Always
          name: watson-reviews
</pre>

### 5. Deploy the application
<!--Deploy in kubernetes (1) nodejs app and (2) couchdb instance. Access via external ip-->

Deploy the CouchDB database. This is where the reviews and Watson Discovery results will be stored.

```
$ kubectl apply -f couchdb.yaml
```

Deploy the Watson Review Analyzer app.

```
$ kubectl apply -f watson-review-analyzer.yaml
```

Check if your Pods are running. You should see a couchdb pod and the watson-review-analyzer pod.

```
$ kubectl get pods

NAME                                         READY     STATUS    RESTARTS   AGE
couchdb-deployment-78c87bf5bf-2nhc2          1/1       Running   0          4d
watson-reviews-deployment-859b8d454f-4zxkw   1/1       Running   0          4d
```

You can access the application in the browser via the Load Balancer's IP (External IP).
If you don't have the Load Balancer enabled, you can access it through one of your Kubernetes' worker IP and service Node port of watson-review-analyzer.

<pre>
$ kubectl get service

NAME             CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
couchdb          172.21.174.52    169.xy.xyz.220   5984:32402/TCP   13d
kubernetes       172.21.0.1       &ltnone&gt           443/TCP          27d
watson-reviews   172.21.254.246   <b>169.48.xyz.221</b>   80:<b>31385</b>/TCP     15m
</pre>

If you don't have the Load Balancer and need the worker IP:
<pre>
$ bx cs workers YOUR_CLUSTER_NAME

OK
ID                                                 Public IP       Private IP       Machine Type   State    Status   Zone    Version
kube-dal12-cr5c30966926aa444f9c02c72d5c3e1ca5-w1   <b>169.xy.xyz.35</b>   10.184.120.196   b2c.16x64      normal   Ready    dal12   1.8.6_1506*
</pre>

Go to `169.xy.xyz.221` or `169.xy.xyz.35:31385` (for clusters without a Load Balancer)

![Landing Page](docs/empty.png)

### 6. Search for a product in Amazon

To use the app, search for the product you want the reviews to be uploaded to Watson Discovery and get its product ID (ASIN).

`https://www.amazon.com/Samsung-Thinnest-Premium-Anti-Scratch-Protective/dp/`**`B06XZ2CM2H`**`/ref=cm_cr_arp_d_product_top?ie=UTF8`

In this case, the product ID is **B06XZ2CM2H**

![Landing Page](docs/analysis.png)

Go ahead and press `Go`.

After Watson Discovery finishes processing all the reviews, the app should show you its General Sentiment and Top entities found.

![Landing Page](docs/concepts.png)

# Privacy Notice

Sample Kubernetes Yaml file that includes this package may be configured to track deployments to [IBM Cloud](https://www.bluemix.net/) and other Kubernetes platforms. The following information is sent to a [Deployment Tracker](https://github.com/IBM/metrics-collector-service) service on each deployment:

* Kubernetes Cluster Provider(`IBM Cloud, Minikube, etc`)
* Kubernetes Cluster ID (Only from IBM Cloud's cluster)

This data is collected from the Kubernetes Job in the sample application's yaml file. This data is used by IBM to track metrics around deployments of sample applications to IBM Cloud to measure the usefulness of our examples so that we can continuously improve the content we offer to you. Only deployments of sample applications that include code to ping the Deployment Tracker service will be tracked.

## Disabling Deployment Tracking

Please comment out/remove the Metric Kubernetes Job portion in the `watson-review-analyzer.yaml` file.

# Links

* [Watson Node.js SDK](https://github.com/watson-developer-cloud/node-sdk)

# Learn more

* **Node.js Code Patterns**: Enjoyed this Code Pattern? Check out our other [Node.js Code Patterns](https://developer.ibm.com/code/technologies/node-js/)
* **Artificial Intelligence Code Patterns**: Enjoyed this Code Pattern? Check out our other [AI Code Patterns](https://developer.ibm.com/code/technologies/artificial-intelligence/).
* **AI and Data Code Pattern Playlist**: Bookmark our [playlist](https://www.youtube.com/playlist?list=PLzUbsvIyrNfknNewObx5N7uGZ5FKH0Fde) with all of our Code Pattern videos
* **With Watson**: Want to take your Watson app to the next level? Looking to utilize Watson Brand assets? [Join the With Watson program](https://www.ibm.com/watson/with-watson/) to leverage exclusive brand, marketing, and tech resources to amplify and accelerate your Watson embedded commercial solution.

# License
[Apache 2.0](LICENSE)
