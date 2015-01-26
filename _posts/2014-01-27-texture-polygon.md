---
layout: post
comments: true
title: Polygon texture bodies with Box2d and cocos2d-x
---

<div class="message">
UPDATE Since I wrote this tutorial cocos2d-x v3 has been released and the code in this tutorial is only for cocos2d-x v2. I might upgrade the code ASAP. Nevertheless the concepts explained in the tutorial should still be relevant. Thank you for reading!
</div>

<div class="message">
UPDATE2 The concepts and mecanics describe here have been implemented in my first game <a href="https://itunes.apple.com/us/app/yummy-jump/id925761778">"Yummy Jump"</a>
</div>

In this tutorial we'll talk about how to create textured physics polygon with [cocos2d-x](http://www.cocos2d-x.org) (2.2.1) and [box2d](http://box2d.org) (2.2.1) using <a href="http://en.wikipedia.org/wiki/Polygon_triangulation">polygon triangulation</a>.
For the impatient the source code is available on <a href="https://github.com/lzubiaur/texpoly" target="_blank">my github</a>.

Basic knowledge of the cocos2d-x framework, box2d and opengl shaders is required.

<iframe width="720" height="315" src="//www.youtube.com/embed/yKfnhK9xknE" frameborder="0" allowfullscreen></iframe>

## Introduction
Box2d is a great middleware physics engine for game. If you are familiar with this engine you might already know that it supports only **convex polygon** with a maximum number of 8 vertices (it might be increased but not recommended).
So if we want to create complex or concave polygon with box2d we have to decompose the body into smaller parts and group them into one big body.

This can be achieved by polygon triangulation where the polygon body is composed of small fixtures triangles.
The image below illustrates how a Box2D concave polygon body with hole is triangulated into several fixtures and how the texture is mapped to render the body.

{% include image.html url="/public/2015/01/drawing.png" description="Triangulate polygon body and texture mapping" %}

The excellent [poly2tri](https://github.com/jhasse/poly2tri) library is used to triangulate the polygons.
Although there's a small drawback using a third party library like poly2tri (we'll have to convert cocos2d point into poly2tri point) it's worth it.
Note that the [clipper](http://www.angusj.com/delphi/clipper.php) library is also included in the project but it's only used to simplify the random generated polygons.

## The big picture

The texture polygons are implemented in the class *TexPoly* which inherit from *CCNode*.
The class definition is shown below but the most important part are the *init* and *draw* methods which will be investigated in depth in the next section.
Put simply a TexPoly object is just a *CCNode* who draws a texture using a shader program. It will also achieve the following tasks.

* Load the texture image
* Triangulate the polygon
* Create the physics box2d body and the fixtures
* Configure the shader program

In order to instanciate a *TexPoly* object we must provide:

* the physics world where the body and fixtures will be created
* the polygon vertices
* optionally a hole vertices
* the texture image file name 

Once the *TexPoly* node is created and added to a layer we have to simulate the physic world by updating both *TexPoly*'s position and rotation.

To do so the Box2D body created by *TexPoly* keeps a reference to the node attached to him (using body's userdata).
It's then possible to iterate through all the bodies and update the *TexPoly* node attached (discussed in the [Simulate the physics world](#SimulatePhysics) section).

{% highlight cpp %}
typedef std::vector<CCPoint&> CCPointVector;

class TexPoly : public CCNode {
public:
    TexPoly();
    virtual ~TexPoly();
    static TexPoly *create(const CCPointVector &points, const std::string &filename, b2World *world);
    static TexPoly *create(const CCPointVector &points, const CCPointVector &hole , const std::string &filename, b2World *world);
    virtual bool init(const CCPointVector &points, const CCPointVector &hole, const std::string &filename, b2World *world);
    virtual void draw();
    void setColor(const ccColor4F &color);
protected:
template <class T> void freeContainer(T &cntr);
protected:
    std::vector<ccVertex3F> mVertexPos;
    std::vector<ccTex2F> mTexCoords;
    GLint mColorLocation;
    ccColor4F mColor;
    CCTexture2D *mTexture;
};
{% endhighlight %}

## Create the polygon
Before we can use the texture we must load the image/pattern into the textures cache.
 The texture will only be loaded once. If it's already been loaded then *addImage* simply returns the texture object from the cache.
 Note that the texture image size must be a power of two (e.g. width=32 and height=32).
 An image with <a href="https://www.opengl.org/wiki/NPOT_Texture">NPOT</a> size will crash badly. You've been warned! ;)

{% highlight cpp %}
mTexture = CCTextureCache::sharedTextureCache()->addImage(filename.c_str());
if (! mTexture) {
    CCLOGERROR("ERROR: Can't load the texture %s", filename.c_str());
    return false;
}
{% endhighlight %}

As written earlier the cocos2d-x CCPoint must been converted into poly2tri (namespace p2t::) points before we can passed them to the triangulation functions. 
The polygon path is converted from CCPoint to p2::Point below.

{% highlight cpp %}
/// The polygon path vertices
std::vector<p2t::Point*> polyline;
  for (const CCPoint&amp; p : points)
    polyline.push_back(new p2t::Point(p.x, p.y));
/// The hole vertices
std::vector<p2t::Point*> holePolyline;   
for (const CCPoint&amp; p : hole)
    holePolyline.push_back(new p2t::Point(p.x, p.y));
{% endhighlight %}

Now we can create the triangulation object called CDT (<a href="http://en.wikipedia.org/wiki/Constrained_Delaunay_triangulation">Constrained Delauney Triangulation</a>) and add the polygon paths (and hole).

Note that the polygon must be a <a href="http://en.wikipedia.org/wiki/Simple_polygon">simple polygon</a> and the points must not repeat!


{% highlight cpp %}
/// Create the CDT instance
p2t::CDT cdt(polyline);
/// Add the hole
    if (holePolyline.size())
cdt.AddHole(holePolyline);
  /// Triangulate!
cdt.Triangulate();
{% endhighlight %}

We can now get the triangulation result from the CDT using GetTriangles() which returns a vector of Triangle object.

{% highlight cpp %}
std::vector<p2t::Triangle*> triangles;
triangles = cdt.GetTriangles();
{% endhighlight %}

From that vector of triangles it's now possible to set up all the fixtures needed to create the body. 
But first things first. We need to instanciate the body before the fixtures. 
Note that the body's user data must be initialized with the address of the TexPoly instance ( *this* pointer).  We'll talk about that later.

{% highlight cpp %}
b2BodyDef bd;
bd.type = b2_dynamicBody;
bd.position = b2Vec2_zero;
bd.userData = this;
b2Body *body = world->CreateBody(&bd);
{% endhighlight %}

OK. Now that we have the body, we just have to create a fixture (of shape *b2PolygonShape*) for each triangles and set the shader vertex (polygon points) and texture coordinates (texture mapping) accordingly.
It's important to note that the fixture's vertices (points) must be converted into meters here (Box2D uses MKS units.
See <a href="http://box2d.org/2011/12/pixels/">this post</a>).

{% highlight cpp %}
/// The box2d polygon shape
b2PolygonShape shape;
/// The fixture definition
b2FixtureDef fd;
fd.density = 1.f;
fd.restitution = .2f;
fd.shape = &shape;
/// The polygon shape's vertices (actually a triangle)
b2Vec2 v[3];
/// Texture size in pixels
CCSize texSize = mTexture->getContentSizeInPixels();
/// Iterate over all triangles
for (p2t::Triangle *tri : triangles) {
/// Iterate over the 3 triangle's points
for (int i = 0; i &lt; 3; ++i) {
    const p2t::Point &p = *tri->GetPoint(i);
    v[i].Set(p.x / PTM_RATIO, p.y / PTM_RATIO);
     mVertexPos.push_back(vertex3(p.x, p.y, .0f));   /// x y z
    mTexCoords.push_back(tex2(p.x / texSize.width, 1 - p.y / texSize.height));
}
/// Create the fixture 
shape.Set(v, 3); 
body->CreateFixture(&fd);
{% endhighlight %}

Before going any further I'd like to say a few words about the previous code. 
When the texture coordinates is outside its range the texture is repeated (actually it's sampled according to its parameters but we'll discuss texture parameters below).
As you probably already know the texture coordinates range from 0 to 1 (where (0,0) is the bottom-left and (1,1) is the top-right corner).
So to map the texture to the polygon we have to convert the polygon's vertices to the texture coordinates unit system.
This is merely what we do in line 20.

Since OpenGL load the texture upside down we have to swap the texture Y coordinate which is done by (1 - Y coordinate).
If you want to know more about texture mapping I encourage you to check those tutorials: <a href="http://en.wikibooks.org/wiki/OpenGL_Programming/Modern_OpenGL_Tutorial_06">OpenGL Tutorial 06</a> and 
<a href="http://open.gl/textures">OpenGL Textures</a>.

Finally we need to configure the shader program for the TexPoly node. Since a CCNode does not have any shader we have to create one.
Cosos2dx offers several shaders for different purpose. I invite you to take a look at the CCGLProgram.h to check the available shaders.

{% highlight cpp %}
/// Enable texture repeat

ccTexParams params = {GL_LINEAR, GL_LINEAR, GL_REPEAT, GL_REPEAT};
mTexture->setTexParameters(&params);
mTexture->retain();

/// Create the shader program
CCGLProgram *program = CCShaderCache::sharedShaderCache()->programForKey(kCCShader_PositionTexture_uColor);
mColorLocation = glGetUniformLocation(program->getProgram(), &quot;u_color&quot;);

setShaderProgram(program);
{% endhighlight %}

In order to tile (repeat) the texture over the polygon surface we have to change the texture wrap mode (line 2 in the code above).
 This is achieved by setting the *GL_TEXTURE_WRAP_S* and *GL_TEXTURE_WRAP_T* texture's parameters to *GL_REPEAT*.
 The other two parameters are both set to *GL_LINEAR* (linear interpolation) which gives a smoother result if the texture is minified (*GL_TEXTURE_MIN_FILTER*) or magnified (*GL_TEXTURE_MAG_FILTER*).


Since we want to be able to communicate the vertex (our polygon points), texture positions and color to the shader we can use the *kCCShader_PositionTexture_uColor* from the shader cache (line 6). 
You can have a look at the shader source code in the *shaders* folder from your cocos2dx distribution.


In line 8 we "get" the color variable from the shader. Because we don't want to update the color we use a uniform shader variable (see the <a href="http://www.opengl.org/sdk/docs/tutorials/ClockworkCoders/uniform.php">OpenGL documentation</a>).
 Use the *kCCShader_PositionLengthTexureColor* shader if you want to vary the color and make gradient effects (e.g. *CCLayerColor*).


## Draw the texture polygon
The texture polygon is drawn inside the TexPoly::draw() method which is just doing the following.

* Bind the texture: tell the shader which texture we want to use (line 8).
* Enable the vertex and texture coordinate parameters (line 10)
* Set the uniform color variable (line 13)
* Pass both vertex and texture coord to the shader (line 15 and 17)
* Set the render primitive. In this case GL_TRIANGLES (line 19)

{% highlight cpp %}
void TexPoly::draw()
{
    if (! isVisible()) return;
    /// Setup the OpenGL shader
    CC_NODE_DRAW_SETUP();
    /// Bind the 2D texture
    ccGLBindTexture2D(mTexture->getName());

    /// Enable shader attributes
    ccGLEnableVertexAttribs(kCCVertexAttribFlag_Position | kCCVertexAttribFlag_TexCoords);

    /// Color
    getShaderProgram()->setUniformLocationWith4f(mColorLocation, mColor.r, mColor.g, mColor.b, mColor.a);
    /// Vertex
    glVertexAttribPointer(kCCVertexAttrib_Position, 3, GL_FLOAT, GL_FALSE, sizeof(ccVertex3F), (void*)&amp;mVertexPos[0]);
    /// TexCoords
    glVertexAttribPointer(kCCVertexAttrib_TexCoords, 2, GL_FLOAT, GL_FALSE, sizeof(ccTex2F), (void*)&amp;mTexCoords[0]);

    /// Available mode: GL_TRIANGLES, GL_TRIANGLES_STRIP and GL_TRIANGLE_FAN
    glDrawArrays(GL_TRIANGLES, 0, mVertexPos.size());

    CHECK_GL_ERROR_DEBUG();

    CC_INCREMENT_GL_DRAWS(1);
}
{% endhighlight %}

## Changing the texture color

If we want to change the color of the texture we might simply write a setColor method as below.
{% highlight cpp %}
void TexPoly::setColor(const ccColor4F &amp;color)
{
    mColor = color;
}
/// Change the texture color
myPolytexture->setColor(ccc4f(1.f, .4f, .3f, 1.f));
{% endhighlight %}

{% include image.html url="/public/2015/01/polygon_color2.jpg" description="Change texture color - example 1." %}
{% include image.html url="/public/2015/01/polygon_color3.jpg" description="Change texture color - example 2." %}

<span id="SimulatePhysics" />

## Simulate the physics world

In HelloWorld::init we can now create the box2d physics world and schedule the update method. 
{% highlight cpp %}
/// Create the physics world
mWorld = new b2World(b2Vec2(0, -10));
scheduleUpdate();
{% endhighlight %}

We override the update method and add our code to update the physics world. Since the bodies's user data was initialized with the TexPoly object we can easily change their position and angle (line 11 and 12).

{% highlight cpp %}
void HelloWorld::update(float dt)
{
    b2Body* next = mWorld->GetBodyList();
    while (next) {
        b2Body* body = next;
        if (body->GetType() != b2_staticBody) {
            /// Get the cocos2d node attached to the box2d physics body
            CCNode *node = static_cast&lt;CCNode *>(body->GetUserData());
            /// Update both CCNode's position and rotation
            node->setPosition(ccp(body->GetPosition().x * PTM_RATIO, body->GetPosition().y * PTM_RATIO));
            node->setRotation( -1 * CC_RADIANS_TO_DEGREES(body->GetAngle()));
        }
        next = body->GetNext();
    }
    /// Update the physics world
    mWorld->Step(1.f / 60.f, 8, 3);
    mWorld->ClearForces();
}
{% endhighlight %}

## What next?

You are now welcome to clone the [example project](https://github.com/lzubiaur/texpoly) and have a look at the TexPoly and HelloWorld classes. You might also build the project and test it by yourself. The project includes a random polygon generator and a example of polygon with hole. 

Have questions or suggestions? Do not hesitate to email me at [{{ site.author.email }}](mailto:{{ site.author.email }}) or ask me on [twitter]({{ site.author.url }}).

{% if page.comments %}
{% include disqus.html %}
{% endif %}
