# Hobby GKE setup
The aim here is to build a reasonable, hobby level gke cluster for ~$15 a month cost.
We take advantage of preemptibles for check compute and nginx ingress to avoid provisoning a loadbalancer. 

# Create cluster
Create a cluster in one for the [always free](https://cloud.google.com/free/docs/gcp-free-tier) 
tier locations, this will allow us to create ingress on a stable f1-micro
instance for free.

When creating the cluster create a node pool with the settings:

Default node pool:
* 2x n1-standard vm
    * 15GB disk
    * Machine class: preemprible
 
Ingress node pool:
* 1x f1-micro
    * 15GB disk
    * Machine class: Normal
    * Add taint: type=ingress
    * Add label: type=ingress

The taint and affinity let us ensure we place ingress on this node

## Static IP
Promote the ephemeral IP on the ingress node to a static IP

## Remove logging
Finally, we will remove logging, default GKE logging adds a fluentd 
instance to each node which takes up too much resource on a f1-micro

```shell script
gcloud  container clusters update --logging-service=none <cluster>
```

# Install ingress

Install nginx ingress with the given given values
```shell script
kubectl create namespace nginx-ingress
helm install nginx-ingress stable/nginx-ingress --values nginx-ingress-values.yaml --namespace nginx-ingress
```
There will be no service created but Nginx will be available on the 
ingress node on ports 80 & 443. You may need to add firewall rules to
expose these ports:

```shell script
gcloud compute firewall-rules create nginx-port --allow tcp:80
gcloud compute firewall-rules create nginx-port --allow tcp:443
```

At this point you should be able to access services through the ingress

# TLS
We will install manager to provision and renew certs from let's encrypt:

```shell script
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.10/deploy/manifests/00-crds.yaml

kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.10.0 
```

# Add let's encrypt production issuer

Modify the email value in lets-encrypt-prod-issuer.yaml to be your email address

```shell script
kubectl apply -f lets-encrypt-prod-issuer.yaml
```

# Add DNS entry
Add dns A record pointing to the ingress node IP for the domain you want to deploy to.
In this case: `sample.crswty.com`

# Test

Modify sample-app.yaml so the host values match the dns entry you created.
```shell script
kubectl apply -f sample-app.yaml
```
Because of the annotation in the ingress `certmanager.k8s.io/cluster-issuer: lets-encrypt-prod` a 
certificate with the value of `secretName` should be automatically created.

In 2-3 minutes you should have a running app with a valid cert.