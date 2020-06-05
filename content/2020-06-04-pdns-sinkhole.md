+++
title="Sinkholing with PowerDNS Recursor"
description="Sinkholing with PowerDNS Recursor"
path="2020/06/pdns-sinkhole"
+++

For some time I've been meaning to set up a
[Pi-hole](https://github.com/pi-hole/pi-hole) to drop advertising and
malware sites by using DNS. 
<!-- more -->

One of the things that has stopped me has
been the size of the tech stack that Pi-hole uses - I don't really want
to pick up a webserver, control panel and dnsmasq just to do some fancy
DNS recursion.

I also have a Kubernetes cluster, so I'm gonna work my way through
running [PowerDNS
Recursor](https://doc.powerdns.com/recursor/index.html) in a container
in Kubernetes, and then setting it to sinkhole sites.

## Running PowerDNS Recursor
This is pretty straightforward. I've created a [minimal
Dockerfile](https://github.com/thommay/docker-pdns-recursor) intended
for use with Kubernetes, and put it [on
DockerHub](https://hub.docker.com/r/thommay/pdns_recursor). I used
[buildah](https://buildah.io/) to build the image, since all my nodes
have cri-o on rather than Docker.

Next, I'll run my `pdns_recursor` container in Kubernetes. We'll use a
[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) to ensure the container is running as a pod, and then a
[Service](https://kubernetes.io/docs/concepts/services-networking/service) with a [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) to expose port 53 on both UDP and TCP. The
config for the recursor will live in a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) and get mounted into
the pod as a [Volume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#add-configmap-data-to-a-volume).
I've put all the kubernetes configs I'm using into [a Gist](https://gist.github.com/thommay/80826aae8cc53187c46d7643f364172a), and will
only excerpt the interesting bits below.

So, here's the pod spec for the deployment:
```yaml
    spec:
      volumes:
      - name: recursor-config-volume
        configMap:
          name: recursor-config
      containers:
      - image: docker.io/thommay/pdns_recursor:4.3.1-1
        imagePullPolicy: IfNotPresent
        name: recursor
        ports:
          - containerPort: 53
            protocol: UDP
            name: udp-dns
          - containerPort: 53
            protocol: TCP
            name: tcp-dns
          - containerPort: 8082
            name: metrics
        volumeMounts:
        - name: recursor-config-volume
          mountPath: /srv/config
```

Into the `recursor-config` [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files) goes a very trivial `recursor.conf`:
```
local-address=0.0.0.0, ::
disable-syslog=yes         # so that our logs show up in the pod's logs
webserver=yes              # for prometheus metrics scraping
webserver-address=0.0.0.0 
```

We'll finish off with a pair of [LoadBalancer
services](https://gist.github.com/thommay/80826aae8cc53187c46d7643f364172a#file-recursor-svc-yaml), for TCP and UDP.

With these running, we can use `dig` to test our recursor:
```
> dig Adsatt.go.starwave.com @192.168.1.232 +short
adimages.go.com.edgesuite.net.
a1412.g.akamai.net.
213.123.244.147
213.123.244.138
```

Great, we have a working DNS resolver.

## Sinkholing
Lots of kind people maintain blocklists of unsavoury sites. A good list
of them is maintained on [the Firebog](https://firebog.net/), along with
some basic validation. You'll probably also want to pick up 
[anudeep's permitted list](https://github.com/anudeepND/whitelist).

PowerDNS allows us to [script the recursor](https://doc.powerdns.com/recursor/lua-scripting/index.html) with Lua,
and we'll use this to import the block lists and check the queries
against them.

I wrote a very trivial Rust binary to do this for me: [here's the
source](https://github.com/thommay/blocklister). You give it a list of
blocklists, a permit list and you get two files containing Lua arrays back.

With those two files, we can write a Lua script that reads them, checks
the query against the lists, and manipulates the result accordingly.
Here's the script:
```lua
adservers=newDS()
permitted=newDS()

function preresolve(dq)
  if permitted:check(dq.qname) or (not adservers:check(dq.qname))  then
    return false
  end

  if(dq.qtype == pdns.A) then
    dq:addAnswer(dq.qtype, "127.0.0.1")
  elseif(dq.qtype == pdns.AAAA) then
    dq:addAnswer(dq.qtype, "::1")
  end
  return true
end

adservers:add(dofile("/srv/config/blocklist.lua"))
permitted:add(dofile("/srv/config/permitted.lua"))
```

We'll add that script and the two generated lists to the
`recursor-config` ConfigMap, and we'll append
```
lua-dns-script=/srv/config/adblock.lua
```
to our `recursor.conf`. Once that's done, we'll restart the recursor
with `kubectl rollout restart deployment/recursor`.

Let's run the same `dig` command we used earlier to check that this
works:
```
> dig Adsatt.go.starwave.com @192.168.1.232 +short
127.0.0.1
```
Perfect! By returning `127.0.0.1`, or `localhost`, we block the
offending site. Our browser (or other application) will issue a
request to the machine it's running on, and should get a very
fast negative response back.

## Automating
There's one problem with what we have currently: it'll get out of date
as new adware and malware sites appear. So let's ensure that we get a
daily update of our lists.

First, we're going to run the blocklister I introduced above as an
[InitContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/).
This is a container that runs before the normal
container, and can be used to set up the pod. In our case, it'll
create the block lists. We'll use an `EmptyDir` volume to share the
configuration between the InitContainer and the recursor.

Here's the InitContainer spec to add to our Deployment:
```yaml
      initContainers:
      - name: blocklister
        image: docker.io/thommay/blocklister:latest
        command: ["/usr/local/bin/blocklister","/srv/blocklister/config.toml"]
        volumeMounts:
        - name: blocklister-config-volume
          mountPath: /srv/blocklister
        - name: data
          mountPath: /srv/data
```

Now, every time our deployment starts up, it'll generate the configs we
need. We'll also need to fix `adblock.lua` to point at the freshly generated lists.
For our final trick, we'll use a CronJob to restart the Deployment
at 3am every day.

For this to work, we have to create a ServiceAccount, give it
permissions to restart the Deployment, and then create a CronJob and get
it to use the ServiceAccount. The complete config is in the [Gist](https://gist.github.com/thommay/80826aae8cc53187c46d7643f364172a#file-recursor-restart-yaml), but
let's look at the CronJob:
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: recursor-restart
  namespace: default
spec:
  concurrencyPolicy: Forbid
  schedule: '0 3 * * *' # 3am daily
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600
      template:
        spec:
          serviceAccountName: recursor-restart
          restartPolicy: Never
          containers:
          - name: kubectl
            image: bitnami/kubectl
            command: ["kubectl","rollout","restart","deployment/recursor"]
```
We ensure we're using the `serviceAccountName` we've already configured,
run the Job on a schedule, and then use a containerised copy of
`kubectl` to perform a rolling restart of the deployment.

And that's it - we've got a fast local recursor that's automatically
updated at 3am nightly with the latest blocklists.

## Recap
We've created a containerised version of PowerDNS Recursor, and run it
in Kubernetes. We've then built a tool to generate blocklists in the
format we need, and caused it to be run every night.
We've now got a sinkhole server on our local network, running a tech
stack that we fully control.
