---
layout: post
comments: true
author: Spencer
title: Posting Kubernetes Events to Slack
tags: docker kubernetes containers golang
---

As a follow-on from yesterday's post, I want to chat some more about the things you could do with the k8s-sniffer go app we created. Once we were able to detect pods in the cluster, handler functions were called when a new pod was created or an existing pod was removed. These handler functions were just printing out to the terminal in our last example, but when you start thinking about it a bit more, you could really do anything you want with that info. We could post pod info to some global registry of systems, we could act upon the metadata for the pods in some way, or we could do something fun like post it to Slack as a bot. Which option do you think I chose?

## **Setting Up Slack** ##

In order to properly communicate with Slack, you will need to set up an incoming webhook. 

- Incoming webhooks are an app you add to Slack. You can find the app [here](https://solinea.slack.com/apps/A0F7XDUAZ-incoming-webhooks). 

- Once this is done, you can configure a new hook. In the "Add Configuration" page, simply select the Slack channel you would like to post to.

- On the next page, save the Webhook URL that is supplied to you and edit the information about your bot as necessary. I added a Kubernetes logo and changed his name to "k8s-bot".


## **Posting To Slack** ##

So with our webhook setup, we are now ready to post to our channel when events occur in the Kubernetes cluster. We will achieve this by adding a new function "notifySlack".

- Add the "notifySlack" method to your k8s-sniffer.go file above the "podCreated" and "podDeleted" functions:

{% highlight go %}
func notifySlack(obj interface{}, action string) {
	pod := obj.(*api.Pod)
	
	//Incoming Webhook URL
    url := "https://hooks.slack.com/your/webhook/url"
	
	//Form JSON payload to send to Slack
	json := `{"text": "Pod ` + action + ` in cluster: ` + pod.ObjectMeta.Name + `"}`

    //Post JSON payload to the Webhook URL
	client := http.Client{}

	req, err := http.NewRequest("POST", url, bytes.NewBufferString(json))
	req.Header.Set("Content-Type", "application/json")

	_, err = client.Do(req)
	if err != nil {
		fmt.Println("Unable to reach the server.")
	}

}
{% endhighlight %}

- Update the url variable with your correct Webhook URL.

- Notice that the function takes an interface and a string as input. This allows us to pass in the pod object that is caught by the handlers, as well as a string indicating whether that pod was added or deleted.

- With this method in place, it's dead simple to update our handler functions to call it instead of outputting to the terminal. Update "podCreated" and "podDeleted" to look like the following:

{% highlight go %}
func podCreated(obj interface{}) {
	notifySlack(obj, "created")
}

func podDeleted(obj interface{}) {
	notifySlack(obj, "deleted")
}
{% endhighlight %}

- The full file will now look like:

{% highlight go %}
package main

import (
	"bytes"
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

func notifySlack(obj interface{}, action string) {
	pod := obj.(*api.Pod)
	
	//Incoming Webhook URL
    url := "https://hooks.slack.com/your/webhook/url"
	
	//Form JSON payload to send to Slack
	json := `{"text": "Pod ` + action + ` in cluster: ` + pod.ObjectMeta.Name + `"}`

    //Post JSON payload to the Webhook URL
	client := http.Client{}

	req, err := http.NewRequest("POST", url, bytes.NewBufferString(json))
	req.Header.Set("Content-Type", "application/json")

	_, err = client.Do(req)
	if err != nil {
		fmt.Println("Unable to reach the server.")
	}

}

func podCreated(obj interface{}) {
	notifySlack(obj, "created")
}

func podDeleted(obj interface{}) {
	notifySlack(obj, "deleted")
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



## **Posted Up** ##

Alright, now when we fire up our go application, we will see posts to our channel in Slack. Remember, the first few will happen quickly, as the store of our pods is populated. 

- Run with `go run k8s-sniffer.go`

- View the first few posts to Slack:

<a href="/img/posts/2016-06-08-Posting-Kubernetes-Events-To-Slack/Slack-Bot-First-Run.png">
<img src="/img/posts/2016-06-08-Posting-Kubernetes-Events-To-Slack/Slack-Bot-First-Run.png" style="max-width75%; border:solid 1px;"/>

- Try scaling down an RC to see the delete: `kubectl scale rc test-rc --replicas=0`

<a href="/img/posts/2016-06-08-Posting-Kubernetes-Events-To-Slack/Slack-Bot-Delete.png">
<img src="/img/posts/2016-06-08-Posting-Kubernetes-Events-To-Slack/Slack-Bot-Delete.png" style="max-width75%; border:solid 1px;"/>


Hope this helps!
