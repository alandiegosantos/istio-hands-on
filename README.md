# Istio Hands-on

This repo contains the file used in the B.DevOps presentation *Tweak interaction between services with Istio*

## What do we need

* istioctl
* a kubernetes cluster (minikube or docker-for-desktop)
* a docker environment


*NOTE:* Your kubernetes clusters should have, at least, 4vCPUs and 6GB of memory.

## Setup

### Prepare the service image

We will use a custom docker image to propagate the http requests across the mesh. Since the purpose is to test Istio, the service is as small as possible. Everytime it receives a request, it requests from its dependency.

If you are using minikube:

```
$ eval $(minikube docker-env)
```

To compile it and build the docker image:

```
$ git submodule update --init --recursive
$ cd servicemesh-test-tool
$ make docker 
```

### Install Istio

To install istio in the cluster, use *istioctl*

```
$ istioctl install --set meshConfig.accessLogFile=/dev/stdout
```

Fix the ingress on Istio-Ingressgateway to use Node Port type.
```
$ kubectl delete service -n istio-system istio-ingressgateway
$ kubectl apply -f setup/istio-ingress.yaml
```
This will also set the ports used to reach Istio Ingress Gateway.

### Install Kiali, Jaeger and Prometheus

We will use kiali, prometheus and jaeger to observe the impact of our changes in the traffic. To install it on Istio 1.7, you can use the following commands:

```
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/jaeger.yaml
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/prometheus.yaml
$ kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.7/samples/addons/kiali.yaml
```

### Setup the environment

In this example, we will use Istio sidecar injection, since it is easiest way to use Istio for now. IMO, it is not valid for a production setup, since it uses iptables rules to redirect all traffic IN and OUT of the pod to Envoy.

To enable sidecar injection on namespace default, just add the label *istio-injection=enabled*

```
$ kubectl label ns default istio-injection=enabled
```

### Start the environment

To create the pods for each service used in this hands-on:

```
$ kubectl apply -f setup/setup.yaml
```

Apply the Istio configuration for our setup:

```
$ kubectl apply -f setup/setup-istio-config.yaml
```

This enables mTLS and all routing rules between services.

### Start jaeger and kiali

To open jaeger and kiali, run in different terminals

```
$ istioctl dashboard kiali
# istioctl dashboard jaeger
```