<!-- NOTE: This file is generated from skewer.yaml.  Do not edit it directly. -->

# Redis Multicloud High Availability using Skupper

#### Secure Redis servers across multiple distributed Kubernetes clusters

This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html

#### Contents

* [Overview](#overview)
* [Step 1: Install the Skupper command-line tool](#step-1-install-the-skupper-command-line-tool)
* [Step 2: Set up your Kubernetes cluster](#step-2-set-up-your-kubernetes-cluster)
* [Step 3: Set up your Podman environment](#step-3-set-up-your-podman-environment)
* [Step 4: Create your sites](#step-4-create-your-sites)
* [Step 5: Link your sites](#step-5-link-your-sites)
* [Step 6: Deploy Redis Server and Sentinel](#step-6-deploy-redis-server-and-sentinel)
* [Step 7: Expose Redis Server and Sentinel to Application Network](#step-7-expose-redis-server-and-sentinel-to-application-network)
* [Step 8: Observe the set of Redis server and Sentinel services in each Site](#step-8-observe-the-set-of-redis-server-and-sentinel-services-in-each-site)
* [Step 9: Create Redis services on podman site](#step-9-create-redis-services-on-podman-site)
* [Step 10: Use Redis command line interface to verify master status](#step-10-use-redis-command-line-interface-to-verify-master-status)
* [Step 11: Deploy the wiki-getter service](#step-11-deploy-the-wiki-getter-service)
* [Step 12: Get Wiki content](#step-12-get-wiki-content)
* [Step 13: Force Sentinel failover](#step-13-force-sentinel-failover)
* [Step 14: Verify Wiki content](#step-14-verify-wiki-content)
* [Step 15: Use Redis command line to measure latency of servers from each site](#step-15-use-redis-command-line-to-measure-latency-of-servers-from-each-site)
* [Step 16: Cleaning up](#step-16-cleaning-up)
* [Summary](#summary)
* [Next steps](#next-steps)
* [About this example](#about-this-example)

## Overview

This example deploys a simple highly available Redis architecture with
Sentinel across multiple Kubernetes clusters using Skupper.

In addition to the Redis Server and Sentinel, the example contains
an additional service:

* A wiki-getter service that exposes an `/api/search?query=` endpoint. 
  The server returns the result from the Redis cache if present otherwise
  it will retrieve the query via the `wiki api` and cache the content via
  the Redis primary server.

With Skupper, you can place the Redis primary server in one cluster and 
the replica servers in alternative clusters without requiring that
the servers be exposed to the public internet.

## Step 1: Install the Skupper command-line tool

This example uses the Skupper command-line tool to deploy Skupper.
You need to install the `skupper` command only once for each
development environment.

On Linux or Mac, you can use the install script (inspect it
[here][install-script]) to download and extract the command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

The script installs the command under your home directory.  It
prompts you to add the command to your path if necessary.

For Windows and other installation options, see [Installing
Skupper][install-docs].

[install-script]: https://github.com/skupperproject/skupper-website/blob/main/input/install.sh
[install-docs]: https://skupper.io/install/

## Step 2: Set up your Kubernetes cluster

Open a new terminal window and log in to your cluster.  Then
create the namespace you wish to use and set the namespace on your
current context.

**Note:** The login procedure varies by provider.  See the
documentation for your chosen providers:

* [Minikube](https://skupper.io/start/minikube.html#cluster-access)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html#cluster-access)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html#cluster-access)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html#cluster-access)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html#cluster-access)
* [OpenShift](https://skupper.io/start/openshift.html#cluster-access)

_**West:**_

~~~ shell
# Enter your provider-specific login command
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**East:**_

~~~ shell
# Enter your provider-specific login command
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

_**North:**_

~~~ shell
# Enter your provider-specific login command
kubectl create namespace north
kubectl config set-context --current --namespace north
~~~

## Step 3: Set up your Podman environment

Open a new terminal window and set the `SKUPPER_PLATFORM`
environment variable to `podman`.  This sets the Skupper platform
to Podman for this terminal session.

Use `podman network create` to create the Podman network that
Skupper will use.

Use `systemctl` to enable the Podman API service.

_**Podman West:**_

~~~ shell
export SKUPPER_PLATFORM=podman
podman network create skupper
systemctl --user enable --now podman.socket
~~~

If the `systemctl` command doesn't work, you can try the `podman
system service` command instead:

~~~
podman system service --time=0 unix://$XDG_RUNTIME_DIR/podman/podman.sock &
~~~

## Step 4: Create your sites

A Skupper _site_ is a location where components of your
application are running.  Sites are linked together to form a
Skupper network for your application.

In Kubernetes, use `skupper init` to create a site.  This
deploys the Skupper router and controller.  Then use `skupper
status` to see the outcome.

In Podman, use `skupper init` with the option `--ingress none`
and use `skupper status` to see the result.

**Note:** If you are using Minikube, you need to [start minikube
tunnel][minikube-tunnel] before you run `skupper init`.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

_**West:**_

~~~ shell
skupper init --site-name west
~~~

_Sample output:_

~~~ console
$ skupper init --site-name west
Waiting for LoadBalancer IP or hostname...
Waiting for status...
Skupper is now installed in namespace 'west'.  Use 'skupper status' to get more information.
~~~

_**East:**_

~~~ shell
skupper init --site-name east
~~~

_Sample output:_

~~~ console
$ skupper init --site-name east
Waiting for LoadBalancer IP or hostname...
Waiting for status...
Skupper is now installed in namespace 'east'.  Use 'skupper status' to get more information.
~~~

_**North:**_

~~~ shell
skupper init --site-name north
~~~

_Sample output:_

~~~ console
$ skupper init --site-name north
Waiting for LoadBalancer IP or hostname...
Waiting for status...
Skupper is now installed in namespace 'north'.  Use 'skupper status' to get more information.
~~~

_**Podman West:**_

~~~ shell
skupper init --site-name podman-west --ingress none
~~~

_Sample output:_

~~~ console
$ skupper init --site-name podman-west --ingress none
It is recommended to enable lingering for currentuser, otherwise Skupper may not start on boot.
Skupper is now installed for user 'currentuser'.  Use 'skupper status' to get more information.
~~~

As you move through the steps below, you can use `skupper status` at
any time to check your progress.

## Step 5: Link your sites

A Skupper _link_ is a channel for communication between two sites.
Links serve as a transport for application connections and
requests.

Creating a link requires use of two `skupper` commands in
conjunction, `skupper token create` and `skupper link create`.

The `skupper token create` command generates a secret token that
signifies permission to create a link.  The token also carries the
link details.  Then, in a remote site, The `skupper link
create` command uses the token to create a link to the site
that generated it.

**Note:** The link token is truly a *secret*.  Anyone who has the
token can link to your site.  Make sure that only those you trust
have access to it.

In this example, for the purpose of availability, all sites are linked together.
This is not a requirement as service communications works across intermediate sites
by way of a shortest path traversal.

_**West:**_

~~~ shell
skupper token create ~/west.token --uses 3
~~~

_Sample output:_

~~~ console
$ skupper token create ~/west.token --uses 3
Token written to ~/west.token
~~~

_**East:**_

~~~ shell
skupper token create ~/east.token --uses 2
skupper link create ~/west.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/east.token --uses 2
Token written to ~/east.token
~~~

_**North:**_

~~~ shell
skupper token create ~/north.token --uses 1
skupper link create ~/west.token
skupper link create ~/east.token
~~~

_**Podman West:**_

~~~ shell
skupper link create ~/west.token
skupper link create ~/east.token
skupper link create ~/north.token
~~~

If your terminal sessions are on different machines, you may need
to use `scp` or a similar tool to transfer the token securely.  By
default, tokens expire after a single use or 15 minutes after
creation.

## Step 6: Deploy Redis Server and Sentinel

A _yaml_ file defines the resources for the Redis deployment in each
site. Each contains:

- A Server deployment resource
- A Sentinel deployment resource
- A Redis config map for configuration

** Note ** the `redis-north.yaml` file designates the server to be
primary while the other sites are designated as replica sites to north.

_**West:**_

~~~ shell
kubectl apply -f redis-west.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f redis-west.yaml
deployment.apps/redis-server created
configmap/redis created
deployment.apps/redis-sentinel created
~~~

_**East:**_

~~~ shell
kubectl apply -f redis-east.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f redis-east.yaml
deployment.apps/redis-server created
configmap/redis created
deployment.apps/redis-sentinel created
~~~

_**North:**_

~~~ shell
kubectl apply -f redis-north.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f redis-north.yaml
deployment.apps/redis-server created
configmap/redis created
deployment.apps/redis-sentinel created
~~~

** Note ** the Sentinel deployments in each site use an init container
that waits for the service definitions of the Redis servers in all
sites to exist before execution. This will be satisfied by the next
step.

## Step 7: Expose Redis Server and Sentinel to Application Network

We will skupper expose the server and sentinel deployments in each namespace

_**West:**_

~~~ shell
skupper expose deployment redis-server --address redis-server-west
skupper expose deployment redis-sentinel --address redis-sentinel-west
~~~

_Sample output:_

~~~ console
$ skupper expose deployment redis-server --address redis-server-west
deployment redis-server exposed as redis-server-west

$ skupper expose deployment redis-sentinel --address redis-sentinel-west
deployment redis-sentinel exposed as redis-sentinel-west
~~~

_**East:**_

~~~ shell
skupper expose deployment redis-server --address redis-server-east
skupper expose deployment redis-sentinel --address redis-sentinel-east
~~~

_Sample output:_

~~~ console
$ skupper expose deployment redis-server --address redis-server-east
deployment redis-server exposed as redis-server-east

$ skupper expose deployment redis-sentinel --address redis-sentinel-east
deployment redis-sentinel exposed as redis-sentinel-east
~~~

_**North:**_

~~~ shell
skupper expose deployment redis-server --address redis-server-north
skupper expose deployment redis-sentinel --address redis-sentinel-north
~~~

_Sample output:_

~~~ console
$ skupper expose deployment redis-server --address redis-server-north
deployment redis-server exposed as redis-server-north

$ skupper expose deployment redis-sentinel --address redis-sentinel-north
deployment redis-sentinel exposed as redis-sentinel-north
~~~

## Step 8: Observe the set of Redis server and Sentinel services in each Site

_**West:**_

~~~ shell
kubectl get services
~~~

_Sample output:_

~~~ console
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
redis-sentinel-east    ClusterIP   172.21.112.88    <none>        26379/TCP                      3m37s
redis-sentinel-north   ClusterIP   172.21.67.12     <none>        26379/TCP                      3m35s
redis-sentinel-west    ClusterIP   172.21.6.3       <none>        26379/TCP                      9m4s
redis-server-east      ClusterIP   172.21.241.21    <none>        6379/TCP                       3m45s
redis-server-north     ClusterIP   172.21.194.72    <none>        6379/TCP                       3m42s
redis-server-west      ClusterIP   172.21.236.8     <none>        6379/TCP                       9m8s
~~~

_**East:**_

~~~ shell
kubectl get services
~~~

_Sample output:_

~~~ console
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
redis-sentinel-east    ClusterIP   172.21.243.22    <none>        26379/TCP                      5m10s
redis-sentinel-north   ClusterIP   172.21.233.113   <none>        26379/TCP                      5m8s
redis-sentinel-west    ClusterIP   172.21.179.114   <none>        26379/TCP                      10m
redis-server-east      ClusterIP   172.21.17.63     <none>        6379/TCP                       5m18s
redis-server-north     ClusterIP   172.21.224.55    <none>        6379/TCP                       5m15s
redis-server-west      ClusterIP   172.21.3.180     <none>        6379/TCP                       10m
~~~

_**North:**_

~~~ shell
kubectl get services
~~~

_Sample output:_

~~~ console
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
redis-sentinel-east    ClusterIP   172.21.211.51    <none>        26379/TCP                      5m54s
redis-sentinel-north   ClusterIP   172.21.66.49     <none>        26379/TCP                      5m52s
redis-sentinel-west    ClusterIP   172.21.83.179    <none>        26379/TCP                      11m
redis-server-east      ClusterIP   172.21.54.164    <none>        6379/TCP                       6m2s
redis-server-north     ClusterIP   172.21.138.118   <none>        6379/TCP                       5m59s
redis-server-west      ClusterIP   172.21.224.110   <none>        6379/TCP                       11m
~~~

## Step 9: Create Redis services on podman site

The podman site does not take part in exchange, service have to be created

_**Podman West:**_

~~~ shell
skupper service create redis-server-north 6379 --host-port 6379
skupper service create redis-server-west 6379 --host-port 6380
skupper service create redis-server-east 6379 --host-port 6381
skupper service create redis-sentinel-north 26379 --host-port 26379
skupper service create redis-sentinel-west 26379 --host-port 26380
skupper service create redis-sentinel-east 26379 --host-port 26381
skupper service status
~~~

_Sample output:_

~~~ console
$ skupper service status
Services exposed through Skupper:
├─ redis-sentinel-east:26379 (tcp)
│  ╰─ Host ports:
│     ╰─ ip: * - ports: 26381 -> 26379
├─ redis-sentinel-north:26379 (tcp)
│  ╰─ Host ports:
│     ╰─ ip: * - ports: 26379 -> 26379
├─ redis-sentinel-west:26379 (tcp)
│  ╰─ Host ports:
│     ╰─ ip: * - ports: 26380 -> 26379
├─ redis-server-east:6379 (tcp)
│  ╰─ Host ports:
│     ╰─ ip: * - ports: 6381 -> 6379
├─ redis-server-north:6379 (tcp)
│  ╰─ Host ports:
│     ╰─ ip: * - ports: 6379 -> 6379
╰─ redis-server-west:6379 (tcp)
   ╰─ Host ports:
         ╰─ ip: * - ports: 6380 -> 6379
~~~

## Step 10: Use Redis command line interface to verify master status

Running the `redis-cli` from the podman-west site, attach to the Redis server
and Sentinel to verfy that the redis-server-north is master.

_**Podman West:**_

~~~ shell
redis-cli -p 6379
127.0.0.1:6379> ROLE
127.0.0.1:6379> exit
redis-cli -p 26379
127.0.0.1:26379> sentinel get-master-addr-by-name redis-skupper
127.0.0.1:26379> exit
~~~

_Sample output:_

~~~ console
$ 127.0.0.1:6379> ROLE
1) "master"
2) (integer) 1531796
3) 1) 1) "redis-server-west"
      2) "6379"
      3) "1531796"
   2) 1) "redis-server-east"
      2) "6379"
      3) "1531796"

$ 127.0.0.1:26379> sentinel get-master-addr-by-name redis-skupper
1) "redis-server-north"
2) "6379"
~~~

## Step 11: Deploy the wiki-getter service

We will choose the north namespace to create a wiki-getter deployment
and service. The client in the service will determine the 
Sentinel service to access the current Redis primary server
for query and cache updates.

_**North:**_

~~~ shell
kubectl apply -f wiki-getter.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f wiki-getter.yaml
deployment.apps/wiki-getter created
service/wiki-getter created
~~~

## Step 12: Get Wiki content

Use `curl` to send a request to querty the Wikipedia API via the 
wiki-getter service. Note the *X-Response-Time* header for the initial
query. The application will check the redis cache and if not found
will fetch from the external Wikipedia API. If the content has been 
stored, the applications will provide the response directly.

_**North:**_

~~~ shell
kubectl exec -it deployment/wiki-getter -- curl -f -I --head http://wiki-getter:8080/api/search?query=Boston
kubectl exec -it deployment/wiki-getter -- curl -f -I --head http://wiki-getter:8080/api/search?query=Boston
~~~

_Sample output:_

~~~ console
$ kubectl exec -it deployment/wiki-getter -- curl -f -I --head http://wiki-getter:8080/api/search?query=Boston
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 132099
ETag: W/"20403-PfBW245Yreh1Gm27jHeiM01Wox8"
X-Response-Time: 1545.706ms
Date: Fri, 29 Mar 2024 12:48:53 GMT
Connection: keep-alive
Keep-Alive: timeout=5

$ kubectl exec -it deployment/wiki-getter -- curl -f -I --head http://wiki-getter:8080/api/search?query=Boston
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 132097
ETag: W/"20401-7u+hiY6DPz+D2DHbukm0QE/L82s"
X-Response-Time: 5.847ms
Date: Fri, 29 Mar 2024 12:48:58 GMT
Connection: keep-alive
Keep-Alive: timeout=5
~~~

## Step 13: Force Sentinel failover

Using the Sentinel command, force a failover as if the master was not reachable. This
will result in the promotion of one of the slave Redis servers to master role.

_**Podman West:**_

~~~ shell
redis-cli -p 26379
127.0.0.1:26379> sentinel failover redis-skupper
127.0.0.1:26379> sentinel get-master-addr-by-name redis-skupper
127.0.0.1:26379> exit
~~~

_Sample output:_

~~~ console
$ 127.0.0.1:26379> sentinel failover redis-skupper
OK
(0.66s)

$ 127.0.0.1:26379> sentinel get-master-addr-by-name redis-skupper
1) "redis-server-west"
2) "6379"
~~~

Note that `redis-server-east` may have alternatively been elected master role.

## Step 14: Verify Wiki content

Check that cached content is correctly returned from new master.

_**North:**_

~~~ shell
kubectl exec -it deployment/wiki-getter -- curl -f -I --head http://wiki-getter:8080/api/search?query=Boston
~~~

_Sample output:_

~~~ console
$ kubectl exec -it deployment/wiki-getter -- curl -f -I --head http://wiki-getter:8080/api/search?query=Boston
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 132097
ETag: W/"20401-7u+hiY6DPz+D2DHbukm0QE/L82s"
X-Response-Time: 152.056ms
Date: Fri, 29 Mar 2024 12:50:45 GMT
Connection: keep-alive
Keep-Alive: timeout=5
~~~

## Step 15: Use Redis command line to measure latency of servers from each site

To understand latency in a true multicloud scenario, the redis-cli can be used to
measure the latency of a Redis server in milliseconds from any application network
site.

_**West:**_

~~~ shell
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-west -p 6379 --raw
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-east -p 6379 --raw
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-north -p 6379 --raw
~~~

_Sample output:_

~~~ console
$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-west -p 6379 --raw
0 10 1.41 88

$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-east -p 6379 --raw
76 254 96.40 10

$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-north -p 6379 --raw
33 104 44.00 19
~~~

_**East:**_

~~~ shell
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-west -p 6379 --raw
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-east -p 6379 --raw
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-north -p 6379 --raw
~~~

_Sample output:_

~~~ console
$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-west -p 6379 --raw
79 110 85.45 11

$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-east -p 6379 --raw
0 26 1.28 89

$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-north -p 6379 --raw
113 358 149.14 7
~~~

_**North:**_

~~~ shell
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-west -p 6379 --raw
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-east -p 6379 --raw
kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-north -p 6379 --raw
~~~

_Sample output:_

~~~ console
$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-west -p 6379 --raw
33 103 38.71 21

$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-east -p 6379 --raw
115 161 125.25 8

$ kubectl exec -it deployment/redis-server -- redis-cli --latency -h redis-server-north -p 6379 --raw
0 19 0.88 94
~~~

_**Podman West:**_

~~~ shell
redis-cli --latency -p 6379 --raw
redis-cli --latency -p 6380 --raw
redis-cli --latency -p 6381 --raw
~~~

_Sample output:_

~~~ console
$ redis-cli --latency -p 6379 --raw
62 88 68.85 13

$ redis-cli --latency -p 6380 --raw
28 38 32.88 24

$ redis-cli --latency -p 6381 --raw
110 280 141.43 7
~~~

The sample period output is latency min, max, average over the number of samples. Note, that the sample
outputs provided are actual measures across three public cloud locations (Washington DC, London, and Dallas)

## Step 16: Cleaning up

To remove Skupper and other resource from this exercise, use the
following commands.

_**West:**_

~~~ shell
kubectl delete -f redis-west.yaml
skupper delete
~~~

_**East:**_

~~~ shell
kubectl delete -f redis-east.yaml
skupper delete
~~~

_**North:**_

~~~ shell
kubectl delete -f wiki-getter.yaml
kubectl delete -f redis-north.yaml
skupper delete
~~~

_**Podman West:**_

~~~ shell
skupper delete
~~~

## Summary

This example locates the Redis Server and Sentinel services in different
namespaces, on different clusters.  Ordinarily, this means that they
have no way to communicate unless they are exposed to the public
internet.

Introducing Skupper into each namespace allows us to create a virtual
application network that can connect Redis services in different clusters.
Any service exposed on the application network is represented as a
local service in all of the linked namespaces.

The Redis primary server is located in `north`, but the Redis replica
services in `west` and `east` can "see" it as if it were local.
Redis replica operations take place by service name and Skupper
forwards the requests to the namespace where the corresponding server
is running and routes the response back appropriately.

## Next steps

Check out the other [examples][examples] on the Skupper website.

## About this example

This example was produced using [Skewer][skewer], a library for
documenting and testing Skupper examples.

[skewer]: https://github.com/skupperproject/skewer

Skewer provides utility functions for generating the README and
running the example steps.  Use the `./plano` command in the project
root to see what is available.

To quickly stand up the example using Minikube, try the `./plano demo`
command.
