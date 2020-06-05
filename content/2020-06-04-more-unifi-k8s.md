+++
title="Running the Unifi Controller in Kubernetes Part 2: ext-dns"
description="Running the Unifi Controller in Kubernetes Part 2: ext-dns"
path="2020/06/more-unifi-k8s-ext-dns"
draft=true
+++

In [Running the Unifi Controller in Kubernetes](@/2019-12-08-unifi-controller-in-k8s.md) I wrote about the process
of getting a Unifi controller running Kubernetes. In this follow up,
I'll add some polish and quality of life improvements. 
<!-- more -->

## The state of play
A quick recap: we have a standalone MongoDB service running, and our
Controller is connected to it. We're using [MetalLB](https://metallb.universe.tf/) to provide layer 2
[LoadBalancers](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer), and we've exposed the controller through one by passing
requests through. The controller UI is available at
[https://192.168.1.231:8443](https://192.168.1.231:8443), and we've
exposed the discovery, controller and stun services as well.

## The plan
I'd like to access my controller at [https://unifi.home.may.lt](https://unifi.home.may.lt), meaning that I need:
 * a DNS record created for `unifi.home.may.lt`
 * a TLS certificate for `unifi.home.may.lt`
 * some form of proxy from port 443 on the front end to 8443 on the
   controller pod

That seems like a good list. Let's go!

### A side note
I originally intended to use [Istio](https://istio.io) on this cluster, because we're
looking at it at work and it seemed like a good learning experience. And
it does (mostly) work, but there are so many moving parts, some weird
bugs/missing features, and because, ironically, the observability of Istio
itself is somewhat terrible, it's a tonne of work to keep things working.
So I ripped Istio back out.

## DNS
Thinking about this from a
[TDD](https://en.wikipedia.org/wiki/Test-driven_development)
perspective, the minimum work to turn this feature green would be to put
an `A` record into my existing DNS to map `unifi.home.may.lt` to
`192.168.1.231`. But I know that I'm gonna add more services to my
cluster, so let's do something a bit more in-depth.

The more in-depth thing we're going to do is
[ExternalDNS](https://github.com/kubernetes-sigs/external-dns#externaldns). 
