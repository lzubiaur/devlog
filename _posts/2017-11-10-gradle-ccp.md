---
layout: post
comments: true
title: Build C++ with Gradle
---

Gradle now supports C/C++ and Objective-C sources.

To bootstrap a C++ project you have to create the gradle wrapper first. Docker is used to
avoid installing gradle ourself.

{% highlight shell %}
docker run --rm -v "$PWD":/home/gradle/project -w /home/gradle/project gradle gradle wrapper
{% endhighlight %}

The command above should have created the following files.

{% highlight shell %}
gradle
gradle
gradlew.bat
{% endhighlight %}

Below a basic `build.gradle` sample. We use the cpp plugin to defined a cpp project.
The project executable is represented using `NativeExecutableSpec`.

{% highlight gradle %}
apply plugin: "cpp"

model {
  components {
    myapp(NativeExecutableSpec) {
      sources {
        cpp {
          source {
            srcDir "src"
            include "**/*.cpp"
          }
        }
      }
    }
  }
}
{% endhighlight %}

The project is built using `gradlew` (use `gradlew.bat` on Windows).

{% highlight shell %}
./gradlew build
{% endhighlight %}

The executable should be available in `build/exe/myapp`
