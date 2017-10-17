```
██╗███████╗████████╗██╗ ██████╗       ██╗      █████╗ ██████╗
██║██╔════╝╚══██╔══╝██║██╔═══██╗      ██║     ██╔══██╗██╔══██╗
██║███████╗   ██║   ██║██║   ██║█████╗██║     ███████║██████╔╝
██║╚════██║   ██║   ██║██║   ██║╚════╝██║     ██╔══██║██╔══██╗
██║███████║   ██║   ██║╚██████╔╝      ███████╗██║  ██║██████╔╝
╚═╝╚══════╝   ╚═╝   ╚═╝ ╚═════╝       ╚══════╝╚═╝  ╚═╝╚═════╝
```

# Introduction to istio

### About

This aims to be a simple intro to what istio is capable of, this the demo of a talk
given in the Barcelona Kubernetes Meetup of June.

Most of the content has been extracted from https://istio.io/docs
Some specific commands  for openshift have been extracted from https://github.com/debianmaster/openshift-examples/tree/master/istio

you can contact me at:

-   jmprusi@(redhat.com|keepalive.io)
-   twitter @jmprusi

### What is this document trying to accomplish

A high level overview of istio and some basic functionality

1. Start an Openshift cluster
2. Install Istio Service Mesh and addons
3. Deploy a sample app (BookInf)
4. Creating a default rule in istio
5. Look at metrics provided by Istio
6. Example of how to do some A/B testing or Beta testing
7. Adding Failures to a service and migrating traffic
8. Adding a rate limit
9. Showing retry capabilities of istio

# Setting up your openshfit cluster

### Install openshift origin client tools

```
Linux 32bit:
- https://github.com/openshift/origin/releases/download/v1.5.1/openshift-origin-client-tools-v1.5.1-7b451fc-linux-32bit.tar.gz

linux 64bit:
- https://github.com/openshift/origin/releases/download/v1.5.1/openshift-origin-client-tools-v1.5.1-7b451fc-linux-64bit.tar.gz

macOS:
- https://github.com/openshift/origin/releases/download/v1.5.1/openshift-origin-client-tools-v1.5.1-7b451fc-mac.zip

Windows:
- https://github.com/openshift/origin/releases/download/v1.5.1/openshift-origin-client-tools-v1.5.1-7b451fc-windows.zip
```

### Start your openshift Cluster

We are using docker for mac.
Make sure your docker daemon has this parameter: "--insecure-registry 172.30.0.0/16"

Then run:
```
oc cluster up --routing-suffix="apps.127.0.0.1.nip.io"
```

### Login as admin
```
oc login -u system:admin
```

### Allow you to login as admin in the console.
```
oc adm policy add-cluster-role-to-user cluster-admin admin
```

# Install Istio Service Mesh

### Set proper privileges for all the istio components
```
oc project default
oc adm policy add-scc-to-user anyuid  -z default
oc adm policy add-scc-to-user privileged -z default
oc patch scc/privileged --patch '{"allowedCapabilities":["NET_ADMIN"]}'

oc adm policy add-cluster-role-to-user cluster-admin -z default

oc adm policy add-cluster-role-to-user cluster-admin -z istio-pilot-service-account
oc adm policy add-cluster-role-to-user cluster-admin -z istio-ingress-service-account

oc adm policy add-scc-to-user anyuid  -z istio-ingress-service-account
oc adm policy add-scc-to-user privileged -z istio-ingress-service-account

oc adm policy add-scc-to-user anyuid  -z istio-pilot-service-account
oc adm policy add-scc-to-user privileged -z istio-pilot-service-account
```

### Download istio 0.1.6
```
git clone https://github.com/istio/istio
cd istio && git checkout 0.1.6
```

### Deploy istio.
```
oc apply -f install/kubernetes/istio.yaml
```

### Install addons
```
oc apply -f install/kubernetes/addons/prometheus.yaml
oc apply -f install/kubernetes/addons/grafana.yaml
oc apply -f install/kubernetes/addons/servicegraph.yaml
oc apply -f install/kubernetes/addons/zipkin.yaml
```
You will need to modify the yml template for servicegraph, changing the TAG from latest to 0.1.6
### Expose addons

```
oc expose svc servicegraph
oc expose svc grafana
oc expose svc prometheus
oc expose svc zipkin
```

### Install istioctl

```
curl -L https://git.io/getIstio | sh -
export PATH="$PATH:$(pwd)/istio-0.1.6/bin"
cd istio-0.1.6
```

# Deploy sample app

### Deploy bookInfo app

```
oc apply -f <(istioctl kube-inject  -f samples/apps/bookinfo/bookinfo.yaml)
```

### Show diff between the original and the injected one

```
colordiff samples/apps/bookinfo/bookinfo.yaml <(istioctl kube-inject  -f samples/apps/bookinfo/bookinfo.yaml)
```

### Expose istio-ingress

```
oc expose svc istio-ingress
```

### Access the bookinfo app
```
open http://istio-ingress-default.apps.127.0.0.1.nip.io/productpage
```

# Look at metrics provided by Istio

### First, let's generate some Traffic

```
 while true; do curl -o /dev/null -s -w "%{http_code}\n" http://istio-ingress-default.apps.127.0.0.1.nip.io/productpage; done
```

### Check grafana
```
open http://grafana-default.apps.127.0.0.1.nip.io/dashboard/db/istio-dashboard
```

### Check servicegraph
```
open http://servicegraph-default.apps.127.0.0.1.nip.io/dotviz
```

# Creating a default rule in istio.

### Create a default route-rule to point every service to the version v1

```
istioctl create -f samples/apps/bookinfo/route-rule-all-v1.yaml

istioctl get route-rules -o yaml
```

### Check grafana to see how the traffic changes
```
open http://grafana-default.apps.127.0.0.1.nip.io/dashboard/db/istio-dashboard
```

# Example of how to do some A/B testing or Beta testing

### A user wants to be a beta tester of our new reviews v2, route only the user jason to reviews v2
```
istioctl create -f samples/apps/bookinfo/route-rule-reviews-test-v2.yaml
```

### Access the bookinfo app
```
open http://istio-ingress-default.apps.127.0.0.1.nip.io/productpage

login as the user "jason" and refresh.

```

### Check grafana to see how the traffic changes
```
open http://grafana-default.apps.127.0.0.1.nip.io/dashboard/db/istio-dashboard
```

# Adding Failures to a service and migrating traffic

### A bug appeared in that version, let's simulate a delay in reviews v2 for "jason"

```
istioctl create -f samples/apps/bookinfo/destination-ratings-test-delay.yaml
```

### Because of the previous bug, let's move a 50% of the traffic to the reviews v3,
### And remove the jason related rules

```
istioctl replace -f samples/apps/bookinfo/route-rule-reviews-50-v3.yaml
istioctl delete route-rule reviews-test-v2
istioctl delete route-rule ratings-test-delay
```

### Check grafana and see how traffic changes.

```
open http://grafana-default.apps.127.0.0.1.nip.io/dashboard/db/istio-dashboard
```

### Everything looks ok, let's migrate everyone to v3.

```
istioctl replace -f samples/apps/bookinfo/route-rule-reviews-v3.yaml
```

### Check grafana and see how traffic changes.

```
open http://grafana-default.apps.127.0.0.1.nip.io/dashboard/db/istio-dashboard
```

# Adding a rate limit

### Rate Limit the ratings service and see how this affects the system

```
istioctl mixer rule create global ratings.default.svc.cluster.local -f samples/apps/bookinfo/mixer-rule-ratings-ratelimit.yaml

istioctl mixer rule get global ratings.default.svc.cluster.local
```

### Check grafana! the ratings serving should start giving 429.

```
open http://grafana-default.apps.127.0.0.1.nip.io/dashboard/db/istio-dashboard
```

### Remove the rate limit

```
istioctl mixer rule delete global ratings.default.svc.cluster.local
```

# Showing retry capabilities of istio

### Add failures to the rating service, 25% of requests will return 503.

```
cat <<EOF | istioctl replace
type: route-rule
name: ratings-default
spec:
  destination: ratings.default.svc.cluster.local
  precedence: 1
  route:
  - tags:
      version: v1
  httpFault:
    abort:
      percent: 25
      httpStatus: 503
EOF
```

### Add a retry rule, from reviews to ratings.

```
cat <<EOF | istioctl create
type: route-rule
name: retries
spec:
  destination: ratings.default.svc.cluster.local
  match:
    source: reviews.default.svc.cluster.local
  precedence: 10
  httpReqRetries:
    simpleRetry:
      attempts: 5
      perTryTimeout: 2s
EOF
```

### Look at zipkin:

```
http://zipkin-default.apps.127.0.0.1.nip.io/

You can query for: response_code=503
Or just stop the traffic generation, wait a minute and try again.
```

# That's it!

### Misc intersting links
```
https://www.istio.io
https://www.istio.io/docs/reference/commands/istioctl.html
https://www.istio.io/docs/reference/config/traffic-rules/routing-rules.html
https://www.istio.io/docs/reference/config/traffic-rules/destination-policies.html
```
