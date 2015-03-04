---
layout: post
comments: true
title: Extend native cocos2d-x C++ class in Lua
---

*Lua* is not a object-oriented language but it still can implement some OOP concepts quite easily.
To create Lua classes cocos2d-x provides the global Lua function **class** (see extern.lua).

Using the *class* function we can extend any cocos2dx native C++ class
(e.g. Node, Sprite, Scene...). We just have to call *class* and provide a name and a function to instanciate the cocos2dx object as the **super** class.

{% highlight lua %}
-- createMySprite returns the native object cc.Sprite
function createMySprite(filename)
    return cc.Sprite:create(filename)
end

-- Create the class "MySprite"
local MySprite = class('MySprite',createMySprite)

-- Instanciate some "MySprite" objects
MySprite:new('sprite01.png')
MySprite:new('sprite02.png')
{% endhighlight %}

*MySprite* objects are usual Lua tables "bound" to a *Sprite* cocos2dx object. We can use them as any other cocos2dx objects in the cocos2dx api but also
add Lua functions or variables to them like any other Lua table.

{% highlight lua %}
Local sprite = MySprite:new('sprite01.png')

-- Use MySprite as a Lua table
function sprite:foo()
    ...
end

sprite.list = {}

-- use MySprite objects with the cocos2dx api
sprite:setScale(2)
scene:addChild(sprite)
{% endhighlight %}

### Advanced cocos2d Lua class

If you want to override C++ methods like *onEnter*, *onExit* or *update* than you'll have to use the *registerScriptHandler*
and *scheduleUpdateWithPriorityLua* function.

But we can use the *class* function to create classes that implicitly override those C++ methods.

{% highlight lua %}
--- Extend a cocos2d native class
-- @param name The class name
-- @param create Function to instanciate the native object.
-- @return The class
local function cocos2dclass(name,create)
    local cls = class(name,create)
    -- override ctor function. init function is called instead
    cls.ctor = function(self,...)
        -- Schedule the update function if exists
        if cls.update then
            self:scheduleUpdateWithPriorityLua(function(dt) self:update(dt) end,1)
        end
        -- Register the events handler
        self:registerScriptHandler(function(event)
            if self.onEnter and event == 'enter' then
                self:onEnter()
            elseif self.onExit and event == 'exit' then
                self:onExit()
            elseif self.onExitTransitionStart and event == 'exitTransitionStart' then
                self:onExitTransitionStart()
            elseif self.onEnterTransitionFinish and event == 'enterTransitionFinish' then
                self:onEnterTransitionFinish()
            elseif self.onCleanup and event == 'cleanup' then
                self:onCleanup()
            end
        end)
        -- Call user initialize function
        if self.init then self:init(...) end
    end
    return cls
end
{% endhighlight %}

In the *cocos2class* function we create a Lua class and define the *ctor* (aka constructor) function which will be
call internally by the function *class*.

Simply put, when we instanciate a class created with *cocos2dclass* the following calls occurs.

1. the *create* function is called to create the native object
2. the *ctor* function is called and we "override" the update and event methods
3. finally we call the *init* function if exists in the class

Now if a *onEnter* or *onExit* function exists in the class then it will be automatically called. If an *update* function is defined
it will be scheduled too.

## Example

{% highlight lua %}
-- Extend the native cocos2dx Scene class
-- The "create" function must return a native object (e.g. cc.Node, cc.Sprite...)
local Scene = cocos2dclass('Scene',function() return cc.Scene:create() end)

-- Initialize the scene with some parameters
function Scene:init(str,value)
    self.str,self.value = str,value
end

-- The update function will be scheduled implicitly if defined
function Scene:update(dt)
    print(dt)
end

-- The onExit function will be registered implicitly and call once the
-- the scene is removed/destroy
function Scene:onExit()
    print 'onExit'
end

-- Instanciate the class "Scene"
local scene = Scene.new('hello',123)

-- The class can be used as a normal Lua table
self.scene.other = 'other field'
print(self.scene.other)

-- But it still can be used as a normal cocos2dx native object
self.scene:addChild(cc.Node:create())
director:replaceScene(self.scene)
{% endhighlight %}
