---
layout: post
comments: true
title: Build Jekyll websites the easy way
---

Go to your Jekyll project and build using the Jekyll Docker image.
{% highlight shell %}
$ docker run --rm -v "$PWD:/srv/jekyll" -it jekyll/jekyll jekyll build
{% endhighlight %}

You can now serve your static website like below.

{% highlight shell %}
$ docker run --rm -v "$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll jekyll serve
{% endhighlight %}

Your local Jekyll website is now available at `http://localhost:4000/my-website`.
