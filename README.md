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

### Testing it

Kiali uses the Istio configuration and prometheus metrics to estimate services interactions.

So, to be able to see things happening in Kiali, we will use *wrk* to generate some traffic.

```
wrk -c 10 -t 5 -d 1h http://localhost:31481 -H "Host: webapp.default.svc.cluster.local"
```

This will run wrk for 1h, using istio ingress gateway to put traffic into the mesh.

To access the service, you can use service exposed by the Ingress Gateway. You should see an output similar to this one.

```
$ curl http://localhost:31481 -H "Host: webapp.default.svc.cluster.local" -v
> GET / HTTP/1.1
> Host: webapp.default.svc.cluster.local
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< date: Wed, 28 Oct 2020 13:57:23 GMT
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 164
< server: istio-envoy
< transfer-encoding: chunked
<
Welcome v1

Headers:
X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
X-B3-Spanid: [f8407728f8d9e632]
X-Envoy-Attempt-Count: [1]
Content-Length: [0]
X-Envoy-Internal: [true]
X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/webapp;Hash=64b868578295d3a116e97b199c35e4a1abfd3bfc84db368536d3156d8d499c13;Subject="";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account]
X-B3-Parentspanid: [439cc2a9693ff9fc]
X-B3-Sampled: [0]
User-Agent: [curl/7.64.1]
Accept: [*/*]
X-Forwarded-For: [192.168.65.3]
X-Forwarded-Proto: [http]
X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]

Dependency: svc1.default.svc.cluster.local Status: 200
-------
	Welcome

	Headers:
	X-Envoy-Internal: [true]
	X-B3-Sampled: [0]
	Accept: [*/*]
	Accept-Encoding: [gzip]
	X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc1;Hash=7605685ef5447a27962b068694a6a7d6ceb1f8ffcff14f6d1a2b559dc8d06c23;Subject="";URI=spiffe://cluster.local/ns/default/sa/webapp]
	X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
	X-Forwarded-For: [192.168.65.3]
	X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
	X-B3-Parentspanid: [a7c849d45cdba905]
	User-Agent: [curl/7.64.1]
	X-Envoy-Attempt-Count: [1]
	X-Forwarded-Proto: [http]
	Content-Length: [0]
	X-B3-Spanid: [c6ce68b906ec932f]

	Dependency: svc3.default.svc.cluster.local Status: 200
	-------
		Welcome

		Headers:
		X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
		X-B3-Spanid: [9c5febfd8e4d8619]
		X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc2;Hash=0a3a99ef419bf8f30cf7a813efe871ed01f364f92fcf30f9a546a19f766f51ad;Subject="";URI=spiffe://cluster.local/ns/default/sa/svc1]
		X-Envoy-Internal: [true]
		User-Agent: [curl/7.64.1]
		Accept: [*/*]
		X-Forwarded-For: [192.168.65.3]
		X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
		Content-Length: [0]
		X-B3-Sampled: [0]
		Accept-Encoding: [gzip]
		X-Envoy-Attempt-Count: [1]
		X-Forwarded-Proto: [http]
		X-B3-Parentspanid: [399da0bece241928]

		Dependency: svc4.default.svc.cluster.local Status: 200
		-------
			Welcome

			Headers:
			X-Envoy-Attempt-Count: [1]
			X-Forwarded-For: [192.168.65.3]
			X-Envoy-Internal: [true]
			X-B3-Spanid: [ab562f0585f72816]
			Accept-Encoding: [gzip]
			X-B3-Sampled: [0]
			User-Agent: [curl/7.64.1]
			X-Forwarded-Proto: [http]
			X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
			X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc4;Hash=6ec8c6ad5fd2267b9b2cee97aac3282b1f127c06ff60c38a4f7af4ebc003db49;Subject="";URI=spiffe://cluster.local/ns/default/sa/svc2]
			X-B3-Parentspanid: [442859d5f6e47e93]
			Accept: [*/*]
			Content-Length: [0]
			X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]

			Processed by svc4-v1-64b87ff94f-pj9vh
		-------
		Dependency: svc5.default.svc.cluster.local Status: 200
		-------
			Welcome

			Headers:
			X-Envoy-Attempt-Count: [1]
			X-Forwarded-For: [192.168.65.3]
			X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
			Content-Length: [0]
			X-B3-Parentspanid: [472fe5a55044aee0]
			Accept: [*/*]
			X-B3-Spanid: [035f4b2140c9cc32]
			Accept-Encoding: [gzip]
			X-Forwarded-Proto: [http]
			X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc5;Hash=6ec8c6ad5fd2267b9b2cee97aac3282b1f127c06ff60c38a4f7af4ebc003db49;Subject="";URI=spiffe://cluster.local/ns/default/sa/svc2]
			X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
			User-Agent: [curl/7.64.1]
			X-Envoy-Internal: [true]
			X-B3-Sampled: [0]

			Processed by svc5-v1-58bd98b677-m64cr
		-------
		Processed by svc3-v1-6f49b554f-5wftb
	-------
	Processed by svc1-v1-665d54f485-dzfvs
-------
Dependency: svc2.default.svc.cluster.local Status: 200
-------
	Welcome

	Headers:
	X-Envoy-Internal: [true]
	Accept: [*/*]
	X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
	Accept-Encoding: [gzip]
	X-B3-Parentspanid: [52fade1f231cd340]
	X-B3-Sampled: [0]
	User-Agent: [curl/7.64.1]
	X-Envoy-Attempt-Count: [1]
	X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc2;Hash=7605685ef5447a27962b068694a6a7d6ceb1f8ffcff14f6d1a2b559dc8d06c23;Subject="";URI=spiffe://cluster.local/ns/default/sa/webapp]
	Content-Length: [0]
	X-Forwarded-For: [192.168.65.3]
	X-Forwarded-Proto: [http]
	X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
	X-B3-Spanid: [f6dd98a58c5bc0dd]

	Dependency: svc4.default.svc.cluster.local Status: 200
	-------
		Welcome

		Headers:
		X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc4;Hash=cf7b8ebf95eaf6c551531e7e2ad1f2c8112ba82e206311ace99747479a72a50e;Subject="";URI=spiffe://cluster.local/ns/default/sa/svc2]
		X-B3-Spanid: [24d56e53722031a0]
		X-B3-Sampled: [0]
		Accept: [*/*]
		X-Forwarded-For: [192.168.65.3]
		X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
		Content-Length: [0]
		X-Envoy-Internal: [true]
		X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
		X-B3-Parentspanid: [dd4d7bb9a78c10d5]
		User-Agent: [curl/7.64.1]
		Accept-Encoding: [gzip]
		X-Envoy-Attempt-Count: [1]
		X-Forwarded-Proto: [http]

		Processed by svc4-v1-64b87ff94f-pj9vh
	-------
	Dependency: svc5.default.svc.cluster.local Status: 200
	-------
		Welcome

		Headers:
		Accept: [*/*]
		X-Forwarded-For: [192.168.65.3]
		X-Envoy-Internal: [true]
		X-B3-Traceid: [be0e679f3ad4c7d3439cc2a9693ff9fc]
		X-B3-Spanid: [3dfec118397b6021]
		User-Agent: [curl/7.64.1]
		X-B3-Parentspanid: [e66b3eb46be90917]
		X-B3-Sampled: [0]
		Accept-Encoding: [gzip]
		X-Forwarded-Proto: [http]
		X-Envoy-Attempt-Count: [1]
		X-Request-Id: [c3d08667-9ed0-48d6-afc8-82df2206cde6]
		Content-Length: [0]
		X-Forwarded-Client-Cert: [By=spiffe://cluster.local/ns/default/sa/svc5;Hash=cf7b8ebf95eaf6c551531e7e2ad1f2c8112ba82e206311ace99747479a72a50e;Subject="";URI=spiffe://cluster.local/ns/default/sa/svc2]

		Processed by svc5-v1-58bd98b677-m64cr
	-------
	Processed by svc2-v1-5d49d4bfdf-rlmcs
-------
* Connection #0 to host localhost left intact
Processed by webapp-v1-68fc7bbfbf-g7dwh* Closing connection 0
```