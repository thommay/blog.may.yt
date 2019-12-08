+++
title="Running the Unifi Controller in Kubernetes"
description="Running the Unifi Controller in Kubernetes"
path="2019/12/unifi-controller-in-k8s"
+++

Over the past few years, I've built up a small collection of
[Ubiquiti Unifi](https://unifi-network.ui.com/) kit. With some small
exceptions - mostly getting BT's multicast HD content working -
pulling this all together has been a smooth process. But one part has
always left me rather dissatisfied: running the management Controller. 
<!-- more -->

### The Unifi Controller
There are fundamentally two ways to run the Controller. You can buy a
Cloud Key and treat it as a managed device, or you can install the
software (there are [Debian packages
available](https://www.ui.com/download/unifi), amongst others) and run
it yourself. Since I have a home server, I've always run this myself by
creating a virtual machine for it and installing the package.  

The Controller itself is a Java app with an embedded MongoDB server for
data retention. One of the reasons for my dissatisfaction has been this
embedded MongoDB - it seemed like every upgrade of the package would
cause the database to get wedged in new and exciting ways, requiring
restore from backup or frantic searching to figure out the commands to
recover the database.

## The Plan
I already had a minimally running [Kubernetes](https://kubernetes.io)
cluster at home, so the plan is to get from there to having the Unifi
Controller connected to a separate MongoDB instance, all the required
ports routable, and all my Unifi kit reporting in correctly.  

### The starting point
A two node Kubernetes cluster, built with [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/) and running 1.16.1 currently.  Container networking provided by [Weave Net](https://www.weave.works/docs/net/latest/kubernetes/), and completely default. The [control-plane node is untainted](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation), so it can run workloads. 

To minimise the amount of raw YAML I needed to write, I chose to
use [Helm 3](https://helm.sh) to manage individual workloads. Once you
have [Helm installed](https://helm.sh/docs/intro/install/), add the
stable repository, which is where we'll find the charts we need:

```console
>  helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

## Persistent Data
Both MongoDB and the Unifi Controller expect to persist data to disk,
and we'll want that data to last for longer than the lifetime of an
individual pod - so we don't lose all our data when we upgrade the
Controller.  
Kubernetes uses [PersistentVolumeClaims](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) (PVCs) to associate storage with a service. A claim is bound to a PersistentVolume, and a pod then mounts the volume. Storage is defined by StorageClasses, and by default there are none available. 

The
[external-storage](https://github.com/kubernetes-incubator/external-storage) 
project provides some implementations of StorageClasses for common
file servers. I already have an NFS server running, so I'll use the
[nfs-client](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client) StorageClass. There's a Helm chart, so we simply need to provide the details of our NFS server:

```console
> helm install  nfs-pvc stable/nfs-client-provisioner --set nfs.server=192.168.1.91 --set nfs.path=/data/pvc
```
You'll also want to have `libnfs-utils` installed on each node of your
cluster.

Once done, you should see a new StorageClass:
```console
> kubectl get storageclass
NAME         PROVISIONER                                    AGE
nfs-client   cluster.local/nfs-pvc-nfs-client-provisioner   4d13h
```
We'll want to set that StorageClass as the [default](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/), so we don't have to
specify it explicitly each time.
```console
> kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## MongoDB
Now we have some storage available, let's get MongoDB running. Helm makes it easy to inspect the
README associated with a chart, so let's start by doing so: 
```console
> helm show readme stable/mongodb
```

We'll want to create a database for the Controller, and a username and
password. I also chose to turn on Prometheus metrics, leaving me with a
`unifi-mongodb.yaml` like this:
```yaml
mongodbUsername: unifi
mongodbPassword: xxxxx
mongodbDatabase: unifi
metrics:
  enabled: true
```

We'll install this chart with Helm:
```console
> helm install unifi-mongodb stable/mongodb -f unifi-mongodb.yaml
```

We can see that a service, called `unifi-mongodb`, and a pod have been
created:
```
> kubectl get svc,po -l app=mongodb
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/unifi-mongodb   ClusterIP   10.96.155.160   <none>        27017/TCP   4d14h

NAME                                 READY   STATUS    RESTARTS   AGE
pod/unifi-mongodb-5bcbcdbdfc-whccb   1/1     Running   0          4d14h
```

Once the pod is Running, we'll want to use the MongoDB client to connect to
the database, so we can perform some additional config. The instructions
to do so are printed by Helm at the end of install, but I'll recap them
here. First, get the MongoDB root password; it's stored as a
[Secret](https://kubernetes.io/docs/concepts/configuration/secret/):

```console
> export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default unifi-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)
```

Now you can run a `mongodb-client` container and connect to the DB.
Kubernetes automatically creates a host name for each service in the
cluster, so we can use `unifi-mongodb` to connect:

```console
> kubectl run --namespace default mongodb-client --rm --tty -i --restart='Never' --image bitnami/mongodb --command -- mongo admin --host unifi-mongodb   --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD
```

We need to grant additional privileges to the `unifi` user, to match
what the Controller expects. We're going to grant `dbOwner` on `unifi`
and `unifi_stat`, meaning that the Controller can drop the database and
recreate it, which it needs to do when restoring from backup. We also
grant `clusterMonitor` on the `admin` database, allowing the Controller
to run the `serverStatus()` command.

```
# first, make sure we're using the unifi database
> use unifi
# next, grant the correct permissions
> db.grantRolesToUser("unifi", [ 
  { db: "unifi", role: "dbOwner" },
  { db: "unifi_stat", role: "dbOwner" },
  { db: "admin", role: "clusterMonitor" }
  ]);
# now verify
> > db.getUser("unifi");
```

That's Mongo fully configured for our use.

## The Unifi Controller

Let's minimally get the Controller running. We want to tell it to use
our freshly baked MongoDB instance, but that's it. Replace XXX in the
URLs with the password you set for the `unifi` user.

```yaml
mongodb:
  enabled: true
  dbUri: mongodb://unifi:XXX@unifi-mongodb/unifi?authSource=unifi
  statDbUri: mongodb://unifi:XXX@unifi-mongodb/unifi_stat?authSource=unifi
  databaseName: unifi
```

As a quick aside, we must specify `authSource`, because user accounts in
MongoDB are tied to a database, but can then be granted access to other
databases. No, me neither.  

As usual, we'll helm install this:
```console
helm install unifi stable/unifi -f unifi.yaml
```

and after a moment or two, we'll have a running unifi pod. We'll also
have some services:

```console
> kubectl get svc -l app.kubernetes.io/name=unifi
```

You'll notice that the TYPE field is a mix of
[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) and ClusterIP (the default). A ClusterIP sets up an IP in the cluster's `serviceSubnet`, meaning it won't be routable outside the cluster. A NodePort exposes the configured port on each node of the cluster, and then forwards that port to a ClusterIP.

To access the web UI, we'll need to port forward 8443 to the pod running our controller:
```console
kubectl --namespace default port-forward --address 0.0.0.0 unifi-795dc84449-dd66q  8443:8443
```

This works, but is pretty annoying; we'll need to have the
port-forward command running all the time. In cloud environments,
Kubernetes works with the cloud's native load balancers to expose
services. At home, we need to do something slightly different.

### MetalLB
[MetalLB](https://metallb.universe.tf/) provides Kubernetes LoadBalancers on bare metal clusters. For the moment, we'll do the simplest possible thing, and use it in [L2 mode](https://metallb.universe.tf/concepts/layer2/).
First, we'll get it installed:
```console
> kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
```

Then, we'll configure L2 mode:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.210-192.168.1.250
```
We're provisioning a pool of 40 IPs here to use for load balancers.
Now, apply that:

```console
> kubectl apply -f metallb.yaml
```

Now, we can update our unifi config to use a LoadBalancer. We
want to run all the services on the same IP, just to make life
easier.

By default Kubernetes doesn't allow you to put TCP ports
and UDP ports on the same LoadBalancer (apparently because GCP
doesn't support it), but MetalLB does support it, so we'll set
an annotation to allow it.

Append this to your existing `unifi.yaml`:

```yaml
guiService: &lbService
  type: LoadBalancer
  annotations:
    metallb.universe.tf/allow-shared-ip: k8s-ext57
    loadBalancerIP: 192.168.1.231
controllerService: *lbService
stunService: *lbService
discoveryService: *lbService
```
You'll note we're using YAML anchors to save duplicating the
config.[^unifiedFN]

Now, we'll install the new config:
```console
> helm upgrade unifi stable/unifi -f unifi.yaml
```

Using `upgrade` rather than `install` means that the configs get
updated, rather than creating new ones. If all's gone to plan, you
should now see `LoadBalancer`s as the service types:
```console
> kubectl get svc -l app.kubernetes.io/name=unifi
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)           AGE
unifi-controller   LoadBalancer   10.96.34.54     192.168.1.231   8080:31917/TCP    4d20h
```

And you'll be able to access your controller at
[https://192.168.1.231:8443](https://192.168.1.231:8443)!

Thanks for sticking with me, I know this got quite long. But hopefully
the end result was worth it.

[^unifiedFN]: In theory, the `unifiedService` should do this for us, but
  I can't get it to work.
