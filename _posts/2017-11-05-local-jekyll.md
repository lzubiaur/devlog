---
layout: post
comments: true
title: Build Jekyll websites the easy way
---

Building your Jekyll static website is now really easy using Docker. Get [Docker](https://store.docker.com/search?offering=community&q=&type=edition) for your platform and you are done.

Go to your Jekyll project and build using the Jekyll Docker image.

{% highlight shell %}
$ docker run --rm -v "$PWD:/srv/jekyll" -it jekyll/jekyll jekyll build
{% endhighlight %}

You are now ready to serve your static website locally.

{% highlight shell %}
$ docker run --rm -v "$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll jekyll serve
{% endhighlight %}

Your website is now available at `http://localhost:4000/my-website`.
