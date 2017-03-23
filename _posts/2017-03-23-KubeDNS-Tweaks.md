---
layout: post
comments: true
author: Spencer
title: KubeDNS Tweaks for Performance
tags: kubernetes containers dns
---

Hey y'all. Wanted to document some of the stranger bits I've encountered while running Kubernetes with one of my clients. We've finally got some decent sized clusters running in their environment and they're being heavily utilized by developers, as they push new or rewritten services into the cluster. Win! That said, we got some complaints about the network performance these guys were seeing. It sounded like intra-cluster communication was working well, but trying to connect to other systems outside of the cluster or things on the public internet were really slow. Like anywhere between 4-10 seconds to resolve the names. Uh oh. Here's some of what we did to help work around that, as well as how we figured it out.

## **Basics** ##

So we had traditionally just been deploying the official KubeDNS deployment that is part of the Kubernetes repo. Or, rather, we were using the one that Kargo deploys, which is just a copy of the former. We'll still be using that as our basis. It's also important to note that the pod that's deployed is 3 containers: kubedns, dnsmasq, and a sidecar for health checking. The names of these seem to have changed very recently, but just know that the important ones are kubedns and dnsmasq.

The flow is basically this:
- A request for resolution inside the cluster is directed to the kubedns service
- The dnsmasq container is the first that receives the request
- If the request is `cluster.local`, `in-addr.arpa`, or similar, it is forwarded to the kubedns container for resolution.
- If it's something else, dnsmasq container queries the upstream DNS that's present in its /etc/resolv.conf file.

## **Logging** ##

So, while all of the above seemed to be working, it was just slow. The first thing I tried to do was see if queries were making it to the dnsmasq container in a timely fashion. I dumped the logs with `kubectl logs -f -tail 100 -c dnsmasq -n kube-system kubedns-xxxyy`. I noticed quickly that there weren't any logs of interest here:
{% highlight bash %}
dnsmasq[1]: started, version 2.76 cachesize 1000
dnsmasq[1]: compile time options: IPv6 GNU-getopt no-DBus no-i18n no-IDN DHCP DHCPv6 no-Lua TFTP no-conntrack ipset auth no-DNSSEC loop-detect inotify
dnsmasq[1]: using nameserver 127.0.0.1#10053
dnsmasq[1]: read /etc/hosts - 7 addresses
{% endhighlight %}

I needed to enable log-queries:
- You can do this by editing the RC with `kubectl edit rc -n kube-system kubedns`. 
- Update the flags under the `dnsmasq` container to look like the following:
{% highlight bash %}
...
      - args:
        - --log-facility=-
        - --cache-size=1000
        - --no-resolv
        - --server=127.0.0.1#10053
        - --log-queries
...
{% endhighlight %}

- Bounce the replicas with:
{% highlight bash %}
kubectl scale rc -n kube-system kubedns --replicas=0 && \
kubectl scale rc -n kube-system kubedns --replicas=1
{% endhighlight %}

Once the new pod is online you can then dump the logs again. You should see lots of requests flowing through, even on a small cluster.

## **WTF Is That?**##
So now that I had some logs online, I started querying from inside of a pod. The first thing I ran was something like `time nslookup kubedns.kube-system.svc.cluster.local` to just simply look up something internal to the cluster. As soon as I did that, I saw a *TON* of queries and, while it eventually resolved, it was searching every. single. possible. name.
{% highlight bash %}
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.kube-system.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.kube-system.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local.kube-system.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.cluster.local to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.default.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.default.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local.default.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.us-west-2.compute.internal from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.us-west-2.compute.internal to 127.0.0.1
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local.compute.internal from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local.compute.internal to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local.compute.internal is NXDOMAIN
dnsmasq[1]: query[A] kubedns.kube-system.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded kubedns.kube-system.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply kubedns.kube-system.svc.cluster.local is 10.233.0.3
{% endhighlight %}

Once I did this, I tried an exteral name to see similar results and a super slow lookup time:

{% highlight bash %}
dnsmasq[1]: query[A] espn.com.kube-system.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded espn.com.kube-system.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply espn.com.kube-system.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] espn.com.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded espn.com.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply espn.com.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] espn.com.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded espn.com.cluster.local to 127.0.0.1
dnsmasq[1]: reply espn.com.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] espn.com.default.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded espn.com.default.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply espn.com.default.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] espn.com.svc.cluster.local from 10.234.96.0
dnsmasq[1]: forwarded espn.com.svc.cluster.local to 127.0.0.1
dnsmasq[1]: reply espn.com.svc.cluster.local is NXDOMAIN
dnsmasq[1]: query[A] espn.com.us-west-2.compute.internal from 10.234.96.0
dnsmasq[1]: forwarded espn.com.us-west-2.compute.internal to 127.0.0.1
dnsmasq[1]: query[A] espn.com.compute.internal from 10.234.96.0
dnsmasq[1]: forwarded espn.com.compute.internal to 127.0.0.1
dnsmasq[1]: reply espn.com.compute.internal is NXDOMAIN
dnsmasq[1]: query[A] espn.com from 10.234.96.0
dnsmasq[1]: forwarded espn.com to 127.0.0.1
dnsmasq[1]: reply espn.com is 199.181.132.250
{% endhighlight %}

What's happening? **It's the ndots.** KubeDNS is hard coded with an ndots value of 5. This means that any request for resolution that contains fewer than 5 dots will cycle through all of the search domains as well in an attempt to resolve. You can see both of these by dumping the /etc/resolv.conf file from the dnsmasq container:

{% highlight bash %}
$ kubectl exec -ti kubedns-mg3tt -n kube-system -c dnsmasq cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local us-west-2.compute.internal compute.internal
nameserver wwww.xxx.yyy.zzzz
options attempts:2
options ndots:5
{% endhighlight %}

It turns out that this is kind of a known issue within KubeDNS if your google-fu is strong enough to find it. Here's a couple of good links for some context:
- https://github.com/kubernetes/kubernetes/issues/33554
- https://github.com/kubernetes/kubernetes/issues/14051
- https://github.com/kubernetes/kubernetes/issues/27679


## **Duct Tape It!** ##
Okay, so from what I was reading, it looked like there wasn't a good consensus on how to fix this, even though an ndots of 3 would have mostly resolved this issue for us. Or at least sped things up enough that we would have been okay with it. And yet, here we are. So we've got to speed this up somehow.

I started reading a bit more about dnsmasq and how we could avoid searching for all of those domain names when we know they don't exist. Enter the `address` flag. This is a dnsmasq flag that you can use to return a defined IP to any request that matches the listed domains. But, if you don't provide the IP it simply returns an NXDOMAIN very quickly and thus doesn't bother forwarding requests up to kubedns or your upstream nameserver. This wound up being the biggest part of our fix. The only real pain in the butt is that you have to list **all** the domains you want to catch. We gave it a good shot, but I'm sure there's more that could be listed. A minor extra is the `--no-negcache` flag. Because we're sending so many NXDOMAIN responses around, we don't want to cache them because it'll eat our whole cache.

The other big part to consider is the `server` flag. This one allows us to specify for a given domain which DNS server should be queried. This seems to actually have been added into the master branch of Kubernetes now as well.

So here's how to fix it:
- Edit the dnsmasq args to look like the following:
{% highlight bash %}
    - args:
      - --log-facility=-
      - --cache-size=10000
      - --no-resolv
      - --server=/cluster.local/127.0.0.1#10053
      - --server=/in-addr.arpa/127.0.0.1#10053
      - --server=/ip6.arpa/127.0.0.1#10053
      - --server=www.xxx.yyy.zzz
      - --log-queries
      - --no-negcache
      - --address=/org.cluster.local/org.svc.cluster.local/org.default.svc.cluster.local/com.cluster.local/com.svc.cluster.local/com.default.svc.cluster.local/net.cluster.local/net.svc.cluster.local/net.default.svc.cluster.local/com.compute.internal/net.compute.internal/com.us-west-2.compute.internal/net.us-west-2.compute.internal/svc.svc.cluster.local/
{% endhighlight %}
- You may find that you want to add more domains as they are relevant to you. We've got some internal domains in the address block that aren't listed here.
- Notice the last server flag. It should point to your upstream DNS server. You can also supply several of these flags if necessary.
- Also note that you may not need to worry about the `compute.internal` domains unless you're in AWS.
- Bounce the replicas again:
{% highlight bash %}
kubectl scale rc -n kube-system kubedns --replicas=0 && \
kubectl scale rc -n kube-system kubedns --replicas=1
{% endhighlight %}

That's it! Hope this helps someone. It really sped up the request time for us. All requests respond in fractions of a second now it seems. I fought with this for a while, but at least had a chance to learn a bit more about how DNS works both inside and outside of Kubernetes.