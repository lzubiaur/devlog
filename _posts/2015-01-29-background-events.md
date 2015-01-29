---
layout: post
comments: true
title: Catch the cocos2d-x application events in Lua
---

If you're developing your cocos2d-x v3 game in Lua you probably want to handle the enter background and foreground events in the Lua code. And this is actually really straightforward using the [new cocos2d-x event dispatcher](http://www.cocos2d-x.org/wiki/EventDispatcher_Mechanism).

First in you **C++ code** you have to edit the *applicationDidEnterBackground* and *applicationWillEnterForeground* methods form the *AppDelegate* class to send the events.

{% highlight cpp %}
void AppDelegate::applicationDidEnterBackground()
{
    auto director = cocos2d::Director::getInstance();
    director->getEventDispatcher()->dispatchCustomEvent("enterBackground");
}

void AppDelegate::applicationWillEnterForeground()
{
    auto director = cocos2d::Director::getInstance();
    director->getEventDispatcher()->dispatchCustomEvent("enterForeground");
}
{% endhighlight %}

Now in your **Lua code** create a function to handle the events. You can create just one single function and simply check the event name.

{% highlight lua %}
local handler = function(event)
    if event:getEventName() == 'enterBackground' then 
        -- Turn music off...
    else
        -- Turn music on...
    end
end 
{% endhighlight %}


Finally register the handler function with both "enterBackground" and "enterForeground" events and voilà!

{% highlight lua %}
local director = cc.Director:getInstance()

director:getEventDispatcher():addEventListenerWithFixedPriority(
    cc.EventListenerCustom:create("enterBackground",handler),1)

director:getEventDispatcher():addEventListenerWithFixedPriority(
    cc.EventListenerCustom:create("enterForeground",handler),1)
{% endhighlight %}

Cocos2d-x guys have made a great job to improve the Lua integration. Check [this wiki](http://www.cocos2d-x.org/wiki/Lua) for more information regarding Lua integration.
