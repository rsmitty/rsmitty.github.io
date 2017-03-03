---
layout: post
comments: true
author: Spencer
title: Extending Our Slack Bot with Image Search
tags: docker slack go golang
---

As a follow-up to [yesterday's post](https://rsmitty.github.io/Slack-Bot/), I wanted to talk about how I built "baconator" at Solinea. This is a goofy Slack bot that we run internally. He responds to requests by querying Google Images for pictures of bacon and then posts them in the channel. Here's how I did it:

## **Get the Proper Googly Bits** ##

#### **Setup Custom Search** ####
To get access to Google Images, we need to create a custom search. This gives us some keys and info we need to pass later on.

- Head to the [CSE main page](https://cse.google.com/cse/).
- Click "Create A Custom Search Engine"
- Fill out the search website by entering "www.google.com"
- Give a name. I called mine "Google Custom Searcher"
<a href="/img/posts/2017-03-03-Slack-Bot-Image-Search/cse-creation.png">
<img src="/img/posts/2017-03-03-Slack-Bot-Image-Search/cse-creation.png" style="max-width:75%; border:solid 1px;"/>

- Once created, hit the "Public URL" button.
- Take a look at the search bar in the browser and copy the "cx" portion. We'll use it later.
<a href="/img/posts/2017-03-03-Slack-Bot-Image-Search/cx-info.png">
<img src="/img/posts/2017-03-03-Slack-Bot-Image-Search/cx-info.png" style="max-width:75%; border:solid 1px;"/>

#### **Setup an API Key** ####
Once that's done, we now have to create an API key. I believe you have to have a Google Cloud project already created, so this may involve a couple of steps.
- Go to the Google developers console's [project page](https://console.developers.google.com/iam-admin/projects).
- Add a new project with a name of your choosing.
- Head to the credentials page for your project. This should be a url like `https://console.developers.google.com/apis/credentials?project=$YOUR_PROJECT_NAME`
- Once there, you'll generate a new API key with "Create Credentials -> API Key".
- Copy down the API key, we'll need it as well.

Phew, now we actually have the bits we need! Let's get the code together.

## **Update the Code** ##

#### **Respond Function** ####
First, we want to turn to our `respond` function that we created last time. What we want to do first is update our accepted phrases to be bacon related, as well as reach out to our next function, `receiveBacon`. This function will be the one that queries our custom search.

- Update `respond` to look like the following:
{% highlight go %}
func respond(rtm *slack.RTM, msg *slack.MessageEvent, prefix string) {
	var response string
	text := msg.Text
	text = strings.TrimPrefix(text, prefix)
	text = strings.TrimSpace(text)
	text = strings.ToLower(text)

	acceptedPhrases := map[string]bool{
		"hook it up":     true,
		"hit me":         true,
		"bacon me":       true,
		"no pork please": true,
	}

	if acceptedPhrases[text] {
		var baconString string
		if text == "no pork please" {
			baconString = "beef+bacon"
		} else {
			baconString = "bacon"
		}
		response = receiveBacon(baconString)
		rtm.SendMessage(rtm.NewOutgoingMessage(response, msg.Channel))
	}
}
{% endhighlight %}

- Note that we're setting a different search string if we encounter the "no pork please" input. Have to respect the varied diets at Solinea, so we search for "beef bacon" in that case :)

#### **Bring Home the Bacon** ####

Now that we've got the respond function setup, let's add our `receiveBacon` function. We'll also create a `random` function that will simply return a number between a min and max. We'll use this to make sure we're seeing fresh bacon each time!

- Add the two functions. They should look like this:
{% highlight go %}
func random(min, max int) int {
	rand.Seed(time.Now().Unix())
	return rand.Intn(max-min) + min
}

func receiveBacon(baconType string) string {

	cxString := os.Getenv("CX_STRING")
	apiKey := os.Getenv("API_KEY")
	startNum := strconv.Itoa(random(1, 10))
	url := "https://www.googleapis.com/customsearch/v1?cx=" + cxString + "&key=" + apiKey + "&q=" + baconType + "&searchType=image&safe=medium&start=" + startNum
	fmt.Printf(url)
	response, err := http.Get(url)
	if err != nil {
		log.Fatal(err)
	}

	defer response.Body.Close()
	body, err := ioutil.ReadAll(response.Body)
	if err != nil {
		log.Fatal(err)
	}

	var jsonData map[string]interface{}
	if err := json.Unmarshal(body, &jsonData); err != nil {
		log.Fatal(err)
	}

	baconNum := random(0, 9)
	items := jsonData["items"].([]interface{})

	baconData := items[baconNum].(map[string]interface{})
	return baconData["link"].(string)
}
{% endhighlight %}

Alright, let's walk through these functions. Assume that the `receiveBacon` function has been called with a baconType of simply "bacon":
- We grab the custom search and API strings from our environment
- Generate a random number between 1 and 10. This will correspond to the page on Google Images
- Craft our request URL and do an `http.Get` on it
- Once we've got our response, unmarshal the json into the jsonData map
- Pick on of the 10 responses on the page by generating another random.
- Return the image link

Once the image link is returned, our `respond` function simply pops it into Slack so we can enjoy our bacon!

Here's the entire testbot.go file:

{% highlight go %}
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"math/rand"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"

	"github.com/nlopes/slack"
)

func main() {

	token := os.Getenv("SLACK_TOKEN")
	api := slack.New(token)
	api.SetDebug(true)

	rtm := api.NewRTM()
	go rtm.ManageConnection()

Loop:
	for {
		select {
		case msg := <-rtm.IncomingEvents:
			fmt.Print("Event Received: ")
			switch ev := msg.Data.(type) {
			case *slack.ConnectedEvent:
				fmt.Println("Connection counter:", ev.ConnectionCount)

			case *slack.MessageEvent:
				fmt.Printf("Message: %v\n", ev)
				info := rtm.GetInfo()
				prefix := fmt.Sprintf("<@%s> ", info.User.ID)

				if ev.User != info.User.ID && strings.HasPrefix(ev.Text, prefix) {
					respond(rtm, ev, prefix)
				}

			case *slack.RTMError:
				fmt.Printf("Error: %s\n", ev.Error())

			case *slack.InvalidAuthEvent:
				fmt.Printf("Invalid credentials")
				break Loop

			default:
				//Take no action
			}
		}
	}
}

func respond(rtm *slack.RTM, msg *slack.MessageEvent, prefix string) {
	var response string
	text := msg.Text
	text = strings.TrimPrefix(text, prefix)
	text = strings.TrimSpace(text)
	text = strings.ToLower(text)

	acceptedPhrases := map[string]bool{
		"hook it up":     true,
		"hit me":         true,
		"bacon me":       true,
		"no pork please": true,
	}

	if acceptedPhrases[text] {
		var baconString string
		if text == "no pork please" {
			baconString = "beef+bacon"
		} else {
			baconString = "bacon"
		}
		response = receiveBacon(baconString)
		rtm.SendMessage(rtm.NewOutgoingMessage(response, msg.Channel))
	}
}

func random(min, max int) int {
	rand.Seed(time.Now().Unix())
	return rand.Intn(max-min) + min
}

func receiveBacon(baconType string) string {

	cxString := os.Getenv("CX_STRING")
	apiKey := os.Getenv("API_KEY")
	startNum := strconv.Itoa(random(1, 10))
	url := "https://www.googleapis.com/customsearch/v1?cx=" + cxString + "&key=" + apiKey + "&q=" + baconType + "&searchType=image&safe=medium&start=" + startNum
	fmt.Printf(url)
	response, err := http.Get(url)
	if err != nil {
		log.Fatal(err)
	}

	defer response.Body.Close()
	body, err := ioutil.ReadAll(response.Body)
	if err != nil {
		log.Fatal(err)
	}

	var jsonData map[string]interface{}
	if err := json.Unmarshal(body, &jsonData); err != nil {
		log.Fatal(err)
	}

	baconNum := random(0, 9)
	items := jsonData["items"].([]interface{})

	baconData := items[baconNum].(map[string]interface{})
	return baconData["link"].(string)
}

{% endhighlight %}

## **Try It Out** ##

Similar to what we did yesterday, let's rebuild our go binary and then our Docker image:
- Build the go binary with `GOOS=linux GOARCH=amd64 go build` in the directory we created.
- Create the container image: `docker build -t testbot .`
- Run it by adding the new necessary env vars: 
`docker run -ti -e SLACK_TOKEN=xxxxxxxxxxxx -e CX_STRING=11111111:aaaaaa \`
`-e API_KEY=abcdefghijklmnop123 testbot`
- Enjoy your hard earned bacon! You'll notice I renamed my bot @baconator.
<a href="/img/posts/2017-03-03-Slack-Bot-Image-Search/bacon.gif">
<img src="/img/posts/2017-03-03-Slack-Bot-Image-Search/bacon.gif" style="max-width:75%; border:solid 1px;"/>