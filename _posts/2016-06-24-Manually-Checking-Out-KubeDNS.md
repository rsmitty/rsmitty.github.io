---
layout: post
comments: true
author: Spencer
title: Manually Interacting with KubeDNS Records
tags: docker kubernetes containers dns
---

A co-worker of mine was having some issues with KubeDNS in his GKE environment. He was then asking how to see if records had actually been added to DNS and I kind of shrugged (via Slack). But this got me a bit curious. How in the heck *do* you look and see? I thought the answer was at least worth writing down and remembering.

## **It's Just etcd** ##

The KubeDNS pod consists of four containers: etcd, kube2sky, exechealthz, and skydns. It's kind of self-explanatory what each do, but etcd is a k/v store that holds the DNS records, kube2sky takes Kubernetes services and pods and updates etcd, and skydns is, guess what, a DNS server that uses etcd as its backend. So it looks like all roads point to etcd as far as where our records live.

## **Checking It Out** ##
Here's how to look at the records in the etcd container:

- Find the full name of the pod for kube-dns with `kubectl get po --all-namespaces`. It should look like `kube-dns-v11-xxxxx`

- Describe the pod to list the containers with `kubectl describe po kube-dns-v11-xxxxx --namespace=kube-system`. We already know what's there, but it's helpful anyways.

- We will now exec into the etcd container and use it's built-in tools to get the data we want. `kubectl exec -ti --namespace=kube-system kube-dns-v11-xxxxx -c etcd -- /bin/sh`

- Once inside the container, let's list all of the services in the default namespace (I've only got one):
{% highlight bash %}
# etcdctl ls skydns/local/cluster/svc/default

/skydns/local/cluster/svc/default/kubernetes
{% endhighlight %}

- Now, find the key for that service by calling ls again:
{% highlight bash %}
# etcdctl ls skydns/local/cluster/svc/default/kubernetes

/skydns/local/cluster/svc/default/kubernetes/8618524b
{% endhighlight %}

- Finally, we can return the data associated with that key by using the get command!
{% highlight bash %}
# etcdctl get skydns/local/cluster/svc/default/kubernetes/8618524b

{"host":"10.55.240.1","priority":10,"weight":10,"ttl":30,"targetstrip":0}
{% endhighlight %}


## **Other Notes** ##
If you want to also test that things are working as expected inside the cluster, follow the great "How Do I Test If It's Working?" section in the DNS addon repo [here](https://github.com/kubernetes/kubernetes/blob/release-1.2/cluster/addons/dns/README.md)
