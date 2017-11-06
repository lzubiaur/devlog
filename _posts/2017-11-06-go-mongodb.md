---
layout: post
comments: true
title: Go and MongoDB in Docker
---

There are many reasons why using containers for local development is very helpful.
One of the many benefits is the availability of containers images for most of the modern software technologies.
There is no more need to spend hours installing softwares and dependencies. Just download the container images you need
and you are almost ready to start developing.

In this small tutorial we are looking to connect a Go application to MongoDB using the available Docker images for Go and MongoDB.

MongoDB installation is straightforward.

{% highlight shell %}
$ docker run --name some-mongo -d mongo
{% endhighlight %}

This will pull the MongoDB image and start the database instance `some-mono`.
The instance is now listening for connections on the default port (27017) but we still have to figure out its IP.

By default Docker containers running on the same host can communicate through the *bridge* network. We are able to check the IP assigned to each container using the `docker network` command.

{% highlight shell %}
$ docker network inspect bridge
{% endhighlight %}

{% highlight json %}
"Containers": {
  "3063164d246f59a7ffbd43f8e9837fc9488e15435639dcb5243c2d3cb22e3865": {
    "Name": "some-mongo",
    "EndpointID": "ddad806db718d0993ddfe9a59e2ed2be64547987b2648372eb275d604ff8984c",
    "MacAddress": "02:42:ac:11:00:04",
    "IPv4Address": "172.17.0.4/16",
    "IPv6Address": ""
    },
{% endhighlight %}

The `some-mongo` instance is available at 172.17.0.4.

Now let's code a very simple Go application to connect to that instance using the [mgo driver](http://labix.org/mgo).

{% highlight go %}
package main

import (
  "fmt"
  "gopkg.in/mgo.v2"
)

func main() {
  session, err := mgo.Dial("172.17.0.4:27017")
  if err != nil {
    panic(err)
  }
  fmt.Printf("Connected to MongoDB!\n")
  defer session.Close()
}
{% endhighlight %}

We are now ready to create the Go container for our application.
The following `Dockerfile` will create a container from the Go base image and copy the source files inside the container.

{% highlight docker %}
FROM golang

WORKDIR /go/src/app
COPY . .

RUN go-wrapper download   # "go get -d -v ./..."
RUN go-wrapper install    # "go install -v ./..."

CMD ["go-wrapper", "run"] # ["app"]
{% endhighlight %}

Time to build our app container.

{% highlight shell %}
$ docker build -t golang-mongo .
{% endhighlight %}

The `Dockerfile` above specifies some `RUN` and `CMD` directives.

The first `RUN` directive will run `go get` to download any package dependencies defined by our app.
The second `RUN` will actually build and install the application inside the container.
The `CMD` defines our Go app as the default command to execute when the container starts.

Finally our application container `golan-mongo` is launched as follow.
Note the `--rm` option so the container is removed when the application exits.

{% highlight shell %}
$docker run -it --rm golang-mongo

Connected to MongoDB!
{% endhighlight %}
