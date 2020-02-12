---
layout: post
title:  "Manage Let's Encrypt SSL Certs on Kubernetes with Cert-Manager"
date:   2020-02-11 15:57:05 -0700
categories: kubernetes
---
Here is a quick guide showing how to automatically issue Let's Encrypt SSL certs on your kubernetes services.
<figure class="video_container">
  <iframe width="740" height="435" src="https://www.youtube.com/embed/LH4nLtUpuBI" frameborder="0" allowfullscreen="true"> </iframe>
</figure>

If you find it easier to follow text instructions, I've outlined the steps below.  They don't follow the video exactly
because I wanted to make it easier to follow without visual queues.  The video lets Helm name the services whereas
we below I name them on the command line.   The video uses the Google Cloud Platform dashboard heavily whereas the 
instructions below are purely command line driven using the Google Cloud SDK command line tools.

## Prerequisites
* [gcloud command-line tools](https://cloud.google.com/sdk/docs/#install_the_latest_cloud_tools_version_cloudsdk_current_version) installed and authenticated with your project selected.
- [helm](https://helm.sh/docs/intro/install/)
- A domain name

## Create a Kubernetes Cluster
Within the Google Cloud Platform, navigate to the [Kubernetes cluster list](https://console.cloud.google.com/kubernetes/list) 
and create a new cluster. If you already have a cluster created you can skip this step.

Alternatively you can accomplish this on the command line with

```bash
gcloud beta container clusters create "standard-cluster-1" --zone=us-central1-b
```

Once your Kubernetes cluster is created, connect to it.  If you created your kubernetes cluster
from the GCP dashboard or you used a different name and zone then above you will want to set those
values appropriately in the credential request below.

```bash
gcloud container clusters get-credentials standard-cluster-1 --zone us-central1-b 
```

## Setup Helm
If you're working with a fresh install of Helm you will need to add two chart repositories.  
The first one is the official Helm stable charts and the second is the Jetstack chart repository. 

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add jetstack https://charts.jetstack.io/
helm repo update
```

## Create an Ingress
You'll start by creating a new cluster role binding admin so as not to run in to any permission issues. 

```bash
kubectl create clusterrolebinding YOURNAME-cluster-admin-binding --clusterrole=cluster-admin --user=YOUREMAIL@EXAMPLE.COM
```

After creating the admin you will use Helm to install nginx-ingress which conveniently works well with cert-manager.

```bash
helm install nginx-ingress stable/nginx-ingress
```

Once the Nginx ingress service has been added you can navigate to your [GCP load balancer configuration](https://console.cloud.google.com/net-services/loadbalancing/loadBalancers/list), 
select the newly created load balancer and copy the front-end IP address. 

Alternatively you can use the command line to grab the IP address of your load balancer with the following command. 

```bash
kubectl describe services nginx-ingress | grep LoadBalancer\ Ingress
```

The response:

```bash
LoadBalancer Ingress:     34.67.5.148
```

If you setup your DNS properly you should be able to run the following command and it will respond with your IP address.
You will need to replace *wp.codejet.net* with your domain since that is the domain I'm using for this example.

```bash
dig wp.codejet.net
```

You will look in the 'ANSWER SECTION' of the response for the IP address.

```bash
;; ANSWER SECTION:
wp.codejet.net.		3599	IN	A	34.67.5.148
```

## Setup Cert-Manager
Before using Helm to install the cert manager you must install the CustomResourceDefinition resources.

```bash
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.13.0/deploy/manifests/00-crds.yaml
```

This step is optional but it's convenient to keep your services within their own namespace so we will create
a cert-manager namespace for our cert-manager to be installed in.

```bash
kubectl create namespace cert-manager
```

Now we can install the cert-manager from the Jetstack chart repo.

```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager
```

Now navigate to your GCP Kubernetes Workloads panel and make sure all the Cert-Manager services
have a status of OK with a green check next to them.   

Alternatively, from the command line you can check that the pods are running with the following command. 

```bash
kubectl get pods -n cert-manager
```

This can take a moment.  Once all the pods have an OK status you can go ahead and create your cluster-issuer. 
This provides cert-manager with an issuer from which it can request SSL certificates.

Create a Let's Encrypt Cluster Issuer after changing *YOUREMAIL@EXAMPLE.COM* to your email address. 

```bash
cat <<EOF > cluster-issuer2.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: YOUREMAIL@EXAMPLE.COM
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-private-key
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

Now apply the configuration to your kubernetes cluster.

```bash
kubectl apply -f cluster-issuer.yaml
```

That should be it for cert-manager but if you want a more in-depth install guide you can read more about [installing cert-manager here](https://cert-manager.io/docs/installation/kubernetes/).

## Create a service 
This can be anything served up over HTTP. For the sake of example I will install WordPress because it's a one-liner.

```bash
helm install wordpress stable/wordpress
```

Once the service is up you will then want to create an ingress for it. Before running the below command you will want to 
edit the content and change *YOURDOMAIN.COM* to your domain.   

```bash
cat <<EOF > ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-wordpress
  annotations:
    kubernetes.io/tls-acme: "true"
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
  - hosts:
    - YOURDOMAIN.COM
    secretName: wordpress-tls
  rules:
  - host: YOURDOMAIN.COM
    http:
      paths:
      - path: /
        backend:
          serviceName: wordpress
          servicePort: 80
EOF
```

Now you may apply your ingress configuration.

```bash
kubectl apply -f ingress.yaml
```

After a few moments you should be able to visit your domain prefixed with https://.  


## Debugging

If everything doesn't magically work (which it should. ^_^), then you'll want to 
[browse to your kubernetes workloads](https://console.cloud.google.com/kubernetes/workload), 
select the cert-manager-XXXXXXXXXX-XXXXX service and view the container logs.  There should be spectacular explosions in there.

Alternatively you can list the pods in the cert-manager namespace and use the command line to view the logs as follows.

```bash
kubectl get pods -n cert-manager
```

The output will look something like this:
```bash
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-667f6bb9f8-vzkgn             1/1     Running   0          13m
cert-manager-cainjector-c5fb4757c-bb2wk   1/1     Running   0          13m
cert-manager-webhook-6c96964bd8-557wj     1/1     Running   0          13m
```

To view the logs for the cert-manager pod (ignoring cainjector and webhook) can be done as follows:
```bash
kubectl logs -f cert-manager-667f6bb9f8-vzkgn -n cert-manager
```

I highly suggest performing this activity a few times through before executing it against a live environment.   I ran through
this at least 2 times from beginning to end before encountering a penny of expense via Google Cloud Platform. When
it was all said and done it cost me 0.30USD (Yes, 30 cents) to run through this easily a half dozen times deleting the
kubernetes cluster after every round.

I destroyed the kubernetes cluster that I created while creating this blog post and video so you will not find anything at wp.codejet.net at this time.

If you find a flaw in this post, [open a pull request](https://github.com/CodeJetNet/codejetnet.github.io) to fix it please!

If you have any issue following these instructions and would like some assistance, please feel free to reach out to me on github, twitter or via email.  
All of those contact points can be found in the footer of this site. Cheers! ^_^
