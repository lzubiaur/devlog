---
layout: post
comments: true
title: Bump collision detection example for cocos2d-x
---

[Bump](https://github.com/kikito/bump.lua) is a great **Lua** library for AABB **collision detection**. Bump works exclusively with rectangle shapes which allows to handle [tunneling](http://www.bulletphysics.org/mediawiki-1.5.8/index.php?title=Collision_Detection_and_Physics_FAQ) fast.

Note that if your gameplay requires physics simulation (joints, collision response...) please have a look at physics engines like [box2D](http://www.box2d.org/), [chipmuck](http://chipmunk-physics.net/) or [liquidfun](http://google.github.io/liquidfun/) (fuild simulation).

Bump is really easy to install and to integrate with cocos2d-x. In the example we will see how to create a moving shape and handle collisions and also how to draw the shape using cocos2d-x.

{% include image.html url="/public/2015/02/bump.gif" description="bump collision detection example" %}

First we have to create the "physics" world using the **newWorld** function. We also create the main cocos2d-x scene and the DrawNode object to render the bump shapes.

{% highlight lua %}
-- Include the bump library
local bump = require 'lib.bump'

-- Create the bump world
local world = bump.newWorld(32)

-- Create the main scene
local scene = cc.Scene:create()

-- Create the draw node to render the shapes
local drawNode = cc.DrawNode:create()
scene:addChild(drawNode)

-- Define some colors
local GREEN       = cc.c4f(0,1,0,1)
local GREEN_FILL  = cc.c4f(0,1,0,0.3)
local YELLOW      = cc.c4f(1,1,0,1)
local YELLOW_FILL = cc.c4f(1,1,0,0.3)
local RED         = cc.c4f(1,0,0,1)
local RED_FILL    = cc.c4f(1,0,0,0.3)
{% endhighlight %}

Then we define the player entity (blue rectangle in the animation above). Any bump shape is a Lua table which is added to the world using the *add* function. A bump shape has a position (bottom-left of the rectangle) and a size (width and height).

In order to move the player with collision detection we have to use the *move* function. We also define a *draw* function to render the player using the DrawNode object.


{% highlight lua %}
local player = world:add({
    x = 0, y = 0,
    vx = 0, vy = 0,
    draw = function(self,drawNode)
        local x,y,w,h = world:getRect(self)
        drawNode:drawSolidRect(cc.p(x,y),cc.p(x+w,y+h),RED_FILL)
        drawNode:drawRect(cc.p(x,y),cc.p(x+w,y+h),RED)
    end,
    filter = function(self)
        return 'slide'
    end,
    update = function(self,dt)
        local ax,ay,cols,len = world:move(self,self.x + self.vx * dt,self.y + self.vy * dt,self.filter)
        self.x, self.y = ax, ay
        for i=1,len do 
            cols[1].other.collide = true
        end
    end,
    center = function(self)
        local x,y,w,h = world:getRect(self)
        return cc.p(x +w/2,y+h/2)
    end
},0,0,20,20)
{% endhighlight %}

A layer is also created to register a touch handler to change the player's direction when the user touch the screen.

{% highlight lua %}
-- Create the "touch layer"
local layer = cc.Layer:create()
layer:registerScriptTouchHandler(function(event,x,y)
    if event == 'began' or event == 'moved' then 
        -- Compute the player's new direction
        local px,py = world:getRect(player)
        local p = cc.pMul(cc.pNormalize(cc.pSub(cc.p(px,py),cc.p(x,y))),100)
        -- update the player velocity
        player.vx = -p.x
        player.vy = -p.y
    end
    return true
end)
layer:setTouchEnabled(true)
scene:addChild(layer,-1)
{% endhighlight %}

We can now create the terrain using random generated rectangle. When the player collides with a terrain shape we change the color from green to yellow.  
{% highlight lua %}
local winSize = cc.Director:getInstance():getWinSize()
math.randomseed(os.time())
for i=1,15 do 
    local ground = world:add({
        collide = false,
        draw = function(self,drawNode)
            local x,y,w,h = world:getRect(self)
            drawNode:drawSolidRect(cc.p(x,y),cc.p(x+w,y+h),self.collide and YELLOW_FILL or GREEN_FILL)
            drawNode:drawRect(cc.p(x,y),cc.p(x+w,y+h),self.collide and YELLOW or GREEN)
        end,
        update = function(self,dt)
            -- Nothing to do
        end
    },math.random(winSize.width),math.random(winSize.height),math.random(100),math.random(100))
end
{% endhighlight %}

Finaly we create a *Node* and schedule a handler function to update and draw the bump entities on every frame.

{% highlight lua %}
local node = cc.Node:create()

-- Schedule the "update" function
node:scheduleUpdateWithPriorityLua(function(dt)
    -- Reset and clear the DrawNode object
    drawNode:clear()
    -- Iterate over all the bump items to update and draw them
    local items, len = world:getItems()
    for i=1,len do
        items[i]:update(dt)
        items[i]:draw(drawNode)
    end
end,1)
scene:addChild(node)

-- Run the scene
cc.Director:getInstance():replaceScene(scene)
{% endhighlight %}

