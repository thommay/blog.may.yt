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

