---
layout: post
comments: true
author: Spencer
title: Tapping Into K8s Events for S's and G's
tags: docker kubernetes containers golang
---

Every now and again I get some pretty interesting questions from clients that stick with me. And rarer than that, I have a bit of free time and get a chance to delve into some of these stranger questions and figure out how you would actually accomplish them. Such is the case with  the question "How do we listen to the Kubernetes clusters we're spinning up and add their resources to an internal registry of systems?". Aren't we supposed to not care that much about our pods, and just let Kubernetes work it's magic? Yes! But hey, sometimes you have to do weird stuff in the enterprise...

So I took this question as a bit of an opportunity to learn a bit more about golang, since my only real experience with it was looking through the Kubernetes and Docker Engine repos from time to time. Luckily, I was able to successfully hack together just enough to act on the creation and deletion of pods in my cluster. I thought this might make for an interesting blog post so other folks can see how it's done and how one might extend this to do some more robust things. Also, you should expect this to also be a bit of a golang intro.

## **Learning by Example** ##

Being that I was pretty new to golang, I felt like I needed a good example to get started parsing and learning about. I recalled from a conversation with a colleague that this type of event sniffing is pretty much exactly how KubeDNS works. The [kube2sky](https://github.com/kubernetes/kubernetes/blob/release-1.2/cluster/addons/dns/kube2sky/kube2sky.go) program acts as a bridge between Kubernetes and the SkyDNS containers that run as part of the DNS addon in a deployed cluster. This program looks for the creation of new services, endpoints, and pods and then configures SkyDNS accordingly by pushing changes to etcd. This was a wonderful starting point, but it took me quite a while to grok what was happening and, after doing so, I just wanted to boil this program down to the basics and do something a bit simpler.

## **Hack Away** ##

Let's get started hacking on our k8s-sniffer program.

- Create a file called k8s-sniffer.go on your system under `$GOPATH/src/k8s-sniffer`. I'm going to operate under the assumption that you've got go already installed.

- Let's add the absolute basics for a standard go program: package, imports, and main function definition

{% highlight go %}
package main

import(
//Import necessary external packages
)

func main(){
//Implement main function
}
{% endhighlight %}

- We've got the bare bones, now let's look at importing the thing's we'll actually need from Kubernetes' go packages. Update your import section to look like:

{% highlight go %}
import (
	"fmt"
	"log"
	"net/http"
	"time"

	"k8s.io/kubernetes/pkg/api"
	"k8s.io/kubernetes/pkg/client/cache"
	"k8s.io/kubernetes/pkg/client/restclient"
	client "k8s.io/kubernetes/pkg/client/unversioned"
	"k8s.io/kubernetes/pkg/controller/framework"
	"k8s.io/kubernetes/pkg/fields"
	"k8s.io/kubernetes/pkg/util/wait"
)
{% endhighlight %}

- Notice the imports at the top look different that the bottom. This is because the ones at the top are golang built-ins. The second ones are from github repositories and go will pull them down for you.

- Go ahead and pull down these dependencies (it'll take a while) by running `go get -v` in the directory containing k8s-sniffer.go

- Now let's get started hacking on the main function. After looking through kube2sky, I knew that I needed to do three things in my main, authenticate to the cluster, call a watcher function, and keep my service alive. You can do this by updating main to look like:
{% highlight go %}
func main() {

	//Configure cluster info
	config := &restclient.Config{
		Host:     "https://xxx.yyy.zzz:443",
		Username: "kube",
		Password: "supersecretpw",
		Insecure: true,
	}

	//Create a new client to interact with cluster and freak if it doesn't work
	kubeClient, err := client.New(config)
	if err != nil {
		log.Fatalln("Client not created sucessfully:", err)
	}

	//Create a cache to store Pods
	var podsStore cache.Store

	//Watch for Pods
	podsStore = watchPods(kubeClient, podsStore)

	//Keep alive
	log.Fatal(http.ListenAndServe(":8080", nil))

}
{% endhighlight %}

- Notice above that some of the configs need to be changed to match your own environment.

- Also notice that many of the functions we're using in this main function come from other packages we've imported.

- If you were to run this program now, the compiler would complain about the fact that you have told it to use the `watchPods` function, but it doesn't actually exist yet. Create this function above main:

{% highlight go %}
func watchPods(client *client.Client, store cache.Store) cache.Store {

	//Define what we want to look for (Pods)
	watchlist := cache.NewListWatchFromClient(client, "pods", api.NamespaceAll, fields.Everything())

	resyncPeriod := 30 * time.Minute

	//Setup an informer to call functions when the watchlist changes
	eStore, eController := framework.NewInformer(
		watchlist,
		&api.Pod{},
		resyncPeriod,
		framework.ResourceEventHandlerFuncs{
			AddFunc:    podCreated,
			DeleteFunc: podDeleted,
		},
	)

	//Run the controller as a goroutine
	go eController.Run(wait.NeverStop)
	return eStore
}
{% endhighlight %}

- And finally, in this function, you'll notice that there are two handler functions called when the watchlist is updated. Create `podCreated` and `podDeleted`:

{% highlight go %}
func podCreated(obj interface{}) {
	pod := obj.(*api.Pod)
	fmt.Println("Pod created: "+pod.ObjectMeta.Name)
}

func podDeleted(obj interface{}) {
	pod := obj.(*api.Pod)
	fmt.Println("Pod deleted: "+pod.ObjectMeta.Name)
}
{% endhighlight %} 

- The full file now looks like:

{% highlight go %}
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"

	"k8s.io/kubernetes/pkg/api"
	"k8s.io/kubernetes/pkg/client/cache"
	"k8s.io/kubernetes/pkg/client/restclient"
	client "k8s.io/kubernetes/pkg/client/unversioned"
	"k8s.io/kubernetes/pkg/controller/framework"
	"k8s.io/kubernetes/pkg/fields"
	"k8s.io/kubernetes/pkg/util/wait"
)

func podCreated(obj interface{}) {
	pod := obj.(*api.Pod)
	fmt.Println("Pod created: "+pod.ObjectMeta.Name)
}

func podDeleted(obj interface{}) {
	pod := obj.(*api.Pod)
	fmt.Println("Pod deleted: "+pod.ObjectMeta.Name)
}

func watchPods(client *client.Client, store cache.Store) cache.Store {

	//Define what we want to look for (Pods)
	watchlist := cache.NewListWatchFromClient(client, "pods", api.NamespaceAll, fields.Everything())

	resyncPeriod := 30 * time.Minute

	//Setup an informer to call functions when the watchlist changes
	eStore, eController := framework.NewInformer(
		watchlist,
		&api.Pod{},
		resyncPeriod,
		framework.ResourceEventHandlerFuncs{
			AddFunc:    podCreated,
			DeleteFunc: podDeleted,
		},
	)

	//Run the controller as a goroutine
	go eController.Run(wait.NeverStop)
	return eStore
}

func main() {

	//Configure cluster info
	config := &restclient.Config{
		Host:     "https://xxx.yyy.zzz:443",
		Username: "kube",
		Password: "supersecretpw",
		Insecure: true,
	}

	//Create a new client to interact with cluster and freak if it doesn't work
	kubeClient, err := client.New(config)
	if err != nil {
		log.Fatalln("Client not created sucessfully:", err)
	}

	//Create a cache to store Pods
	var podsStore cache.Store

	//Watch for Pods
	podsStore = watchPods(kubeClient, podsStore)

	//Keep alive
	log.Fatal(http.ListenAndServe(":8080", nil))
}
{% endhighlight %}

## **Fire Away** ##

- We can finally run our file and see events being created when new Pods are created or destroyed! You'll see several alerts when you first run since the pods are getting added to the store.

{% highlight bash %}
spencers-mbp:k8s-siffer spencer$ go run k8s-sniffer.go
Pod created: dnsmasq-vx2sw
Pod created: default-http-backend-0zj29
Pod created: nginx-ingress-lb-xgvin
Pod created: kubedash-3370066188-rmy2n
Pod created: dnsmasq-gru7c
Pod created: kubernetes-dashboard-imtnm
Pod created: kube-dns-v11-dhgyx
Pod created: test-rc-h7v6l
Pod created: test-rc-3l1oo
{% endhighlight %} 


- Try scaling down an RC to see the delete: `kubectl scale rc test-rc --replicas=0`

{% highlight bash %}
Pod deleted: test-rc-h7v6l
Pod deleted: test-rc-3l1oo
{% endhighlight %}

Hope this helps!
