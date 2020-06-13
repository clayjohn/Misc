# 2D Batching

## Introduction
In order to tell the GPU what to draw, and where, game engines have to send a set of instructions to the GPU in order to draw a frame. These instructions are sent in common languages, called APIs (application programming interfaces), examples of which are OpenGL, OpenGL ES 2, OpenGL ES3 and Vulkan.

### Drawcalls
In 2D, we need to tell the GPU to render a series of primitives (rectangles, lines, polygons etc). The most obvious technique is to tell the GPU to render one primitive at a time, telling it some information such as the texture used, the material, the position, size etc, then saying 'Draw!' (this is called a drawcall).

It turns out that while this is conceptually simple from the engine side, GPUs operate very slowly when used in this manner. GPUs work much more efficiently if, instead of telling them to draw a single primitive, you tell them to draw a number of similar primitives all in one drawcall, which we will call a 'batch'.

And it turns out that they don't just work a bit faster when used in this manner, they work A LOT faster.

As Godot is designed to be a general purpose engine, the primitives coming into the Godot renderer can be in any order, sometimes related, and sometimes unrelated. In order to match the general purpose nature of Godot with the batching preferences of GPUs, Godot features an intermediate layer which can automatically group together similar primitives wherever possible, and send these batches on to the GPU. This can give an increase in rendering performance while requiring few, if any, changes to your Godot project.

## How it works

Inside the renderer instructions come in from your game in the form of a series of items, each of which can contain one or more commands. The items correspond to Nodes in the scene tree, and the commands correspond to primitives such as rectangles or polygons. Some items, such as tilemaps, and text, can contain a large number of commands (tiles and letters respectively). Others, such as sprites, may only contain a single command (rectangle).

The batcher uses two main techniques to group together primitives:
* Consecutive items can be joined together
* Consecutive commands within an item can be joined to form a batch

### Breaking batching
These can only take place if the items or commands are similar enough to be rendered in one drawcall. Certain changes (or techniques), by necessity, prevent the formation of a contiguous batch, this is referred to as 'breaking batching'.

Batching will be broken by (amongst other things):
* Change of texture
* Change of material
* Change of primitive type (say going from rectangles to lines)

If for example, you draw a series of sprites each with a different texture, there is no way they can be batched.

### Render order
The question arises, if only similar items can be drawn together in a batch, why don't we look through all the items in a scene, group together all the similar items, and draw them together?

In 3D, this is often exactly how engines work. However, in Godot 2D, items are drawn in 'painter's order', from back to front. This ensures that items at the front are drawn on top of earlier items, when they overlap.

This also means that if we try and draw objects in order of, for example, texture, then this painter's order may be broken and the wrong objects end up being drawn at the front.

In Godot this back to front order is determined by:
* The order of objects in the scene tree
* The Z index of objects
* The canvas layer
* Y sort nodes

It should be apparent that there is a certain amount of leeway in your scene design for you to order objects in such a way that similar objects will be grouped together for easier batching. Note that this is not a requirement on your part, think of it as an optional approach that can improve performance in some cases. See the diagnostics section in order to help you make this decision.

### A trick
And now a sleight of hand. Although the idea of painter's order is that objects are rendered from back to front, consider 3 objects A, B and C, that contain 2 different textures, metal and wood.

In painter's order they are ordered:
A - wood
B - metal
C - wood

Because the texture changes, they cannot be batched, and will be rendered in 3 drawcalls.

However, painter's order is needed on the assumption that they will be drawn ON TOP of each other. If we relax that assumption, i.e. if none of these 3 objects are overlapping, there is NO NEED to preserve painter's order. The rendered result will be the same. What if we could take advantage of this?

A - wood
C - wood
B - metal
_(2 drawcalls)_

### Item reordering

It turns out that we can reorder items. However, we can only do this if the items satisfy the conditions of an overlap test, to ensure that the end result will be the same as if they were not reordered. The overlap test is very cheap in performance terms, but not absolutely free, so there is a slight cost to looking ahead to decide whether items can be reordered. The number of items to lookahead for reordering can be set in project settings (see below), in order to balance the costs and benefits in your project.

## Lights

Although the job for the batching system is normally quite straightforward, it becomes considerably more complex when 2D lights are used, because lights are drawn using multiple extra passes, one for each light affecting the primitive. Consider 2 sprites A and B, with identical texture and material. Without lights they would be batched together and drawn in one drawcall. But with 3 lights, they would be drawn as follows, each line a drawcall:

A
A - light 1
A - light 2
A - light 3
B
B - light 1
B - light 2
B - light 3

That is a lot of drawcalls, 8 for only 2 sprites. Now consider we are drawing 1000 sprites, the number of drawcalls quickly becomes astronomical, and performance suffers. This is partly why lights has the potential to drastically slow down 2D.

However, if you remember our magician's trick from item reordering, it turns out we can use the same trick to get around painter's order for lights!

If A and B are not overlapping, we can render them together in a batch, so the draw process is as follows:

AB
AB - light 1
AB - light 2
AB - light 3

That is 4 drawcalls. Not bad, that is a 50% improvement. However consider that in a real game, you might be drawing closer to 1000 sprites.

* Before
1000 * 4 = 4000 drawcalls.
* After
1 * 4 = 4 drawcalls.

That is 1000x decrease in drawcalls, and will usually give a huge increase in performance.

### Overlap test
However, as with the item reordering, things are not that simple, we must first perform the overlap test to determine whether we can join these primitives, and the overlap test has a small cost. So again you can choose the number of primitives to lookahead in the overlap test to balance the benefits against the cost. Usually with lights the benefits far outweigh the costs.

Also consider that depending on the arrangement of primitives in the viewport, the overlap test will sometimes fail (because the primitives overlap and thus should not be joined). So in practice the decrease in drawcalls may be less dramatic than the perfect situation of no overlap. However performance is usually far higher than without the overlap tests.


## Parameters
In order to fine tune batching, a number of project settings are available. You can usually leave these at default during development, but it is a good idea to experiment to ensure you are getting maximum performance. Spending a little time tweaking parameters can often give considerable performance gain, for very little effort.

### Options (in rendering/batching/options/)
use_batching - turns batching on and off
use_batching_in_editor
### Parameters (in rendering/batching/parameters/)
#### max_join_item_commands
One of the most important ways of achieving batching is to join suitable adjacent items (nodes) together, however they can only be joined if the commands they contain are compatible and there is 


## Diagnostics
Although you can change parameters and examine the effect on frame rate, this can feel like working blind, with no idea of what is going on under the hood. To help with this, batching offers a diagnostic mode, which will periodically print out a list of the batches that are being processed. This can help pin point situations where batching is not occurring as intended, and help you to fix them, in order to get the best possible performance.

### Custom Shaders
Batching has been primarily designed to accelerate the common cases found in projects, there are some features which will unexpectedly break, or decrease batching. Many of these are in the use of custom shaders. While we will work to increase the number of cases that can be batched, currently some of following in custom shaders will prevent some batching optimizations:

* Reading or writing COLOR or MODULATE - disables vertex color baking
* Reading VERTEX - disables vertex position baking

## Performance Tuning

### Bottlenecks
While batching is a specific optimization to reduce drawcalls (and state changes) and make better use of the GPU, in terms of overall performance benefit it can only be looked at in the context of where the bottlenecks are in your game or project.

The proverb 'a chain is only as strong as its weakest link' applies directly to performance optimization. If your project is spending 90% of the time in e.g. API housekeeping due to drawcalls / state changes, then reducing this by applying batching can have a massive effect on performance.

API housekeeping 9 ms
Everything else 1 ms
Total : 10 ms

API housekeeping 1 ms
Everything else 1ms
Total : 2 ms

So in this example batching improving this bottleneck by a factor of 9x, decreases overall frame time by 5x, and increases frames per second by 5x.

If however, something else is running slowly and also bottlenecking your project, then the same improvement to batching can lead to less dramatic gains:

API housekeeping 9 ms
Everything else 50 ms
Total : 59 ms

API housekeeping 1 ms
Everything else 50 ms
Total : 51 ms

So in this example, even though we have hugely optimized the batching, the actual gain in terms of frame rate is quite small.

The takehome message is that while batching improves the performance of a certain part of the engine, it is not a magic bullet, and is only a piece in the jigsaw of achieving high performance.

Optimization is thus a continuous process:

* Identify the bottlenecks
* Optimize the slowest bottleneck (low hanging fruit)
* Repeat

Other areas highly likely to be bottlenecks:
* Scripts
* Node updates (large number of nodes)
* GPU fill rate (lots of pixels being shaded and blended can slow the GPU, especially on mobile)

## FAQ

### I don't get a large performance increase from switching on batching
* Try the diagnostics, see how much batching is occurring, and whether it can be improved
* Try changing parameters
* Consider that batching may not be your bottleneck (see bottlenecks)

### I get a decrease in performance with batching
* Try steps to increase batching given above
* Try switching 'single_rect_fallback' to on
* The single rect fallback method is the default used without batching, however it can result in flicker on some hardware, so is discouraged

### I use custom shaders and the items are not batching
* Custom shaders can be problematic for batching, see the custom shaders section

### I use a large number of textures, so few items are being batched
* Consider the use of texture atlases. As well as allowing batching these reduce the need for state changes associated with changing texture.