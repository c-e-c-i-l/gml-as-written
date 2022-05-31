![image](/topics/debugoverlay/img/header.png)

# the debug overlay _!_

## [ scary graph noises intensify ]

So you found `show_debug_overlay()` in the manual, and now you're looking at it and it's moving so fast and all the colors and numbers are confusing and you just want to know why your game is lagging.

I have some bad news. The debug overlay isn't going to be able to tell you why your game is broken, especially if it is broken for shader reasons. It can however, give you a good _idea_ of where to start looking to figure out what broke the game.

## the gist

The debug overlay is a graph and some numbers. It has the following puzzle pieces: your `fps_real`, a graph the amount of time specific processes are taking to resolve, the number of texture swaps your game has, and the number of batch breaks. This data gives you a decently accurate picture of your game's performance state without having to run the game in debug mode which adds its own overhead to the whole process.

![image](/topics/debugoverlay/img/fpsreal.png)

## fps_real

This is perhaps the most misleading thing you can look at to judge your game's performance. First of all, this is a measure of how fast your game could run if it were not bottlenecked by cpu cycle times. To make a complex topic into a thin soup: your game is a constant loop and each iteration of the loop finishes before the cpu cycle does. In order to conserve power, your game sleeps for the remainder of that cycle then wakes up to start the next iteration. If your cpu did not have cycles, and your game did not go to sleep each cycle, this is (sort of) how many frames per second your game would run. Adding or removing things from your game can alter this number misleadingly, and it can also be different based on how many youtube videos you're watching at once. This is an unrealistic thing to measure your game by, and you should instead just babysit this number until it gets to an uncomfortable point.

Personally I treat fps_real like a budget for cool stuff in my game, and just keep adding cool stuff until fps_real gets in between two and five hundred.

![image](/topics/debugoverlay/img/swap.png)

## texture swaps

Texture swaps and batch breaks are things you will see various folks on the internet screaming about keeping low, which can make these numbers scary as they get high. In reality its not as simple as **low number good big number bad ooga ooga**. But! Before you can understand how big this number can be before it's too big, you need to understand what a texture swap is.

When you're fiddlin' about in gm and you add a sprite to your game it gets put on something called a texture page. This is like a big sticker sheet of your game's sprites. Each page can only hold so many sprites, so when you add more and more sprites you end up with multiple texture pages. When your game goes to draw something, the cpu passes the texture page to the gpu, along with information about what on the texture page needs to be drawn.

Well the gpu is very good at its job. If you have a hundred sprites to be drawn, and they're all on the same texture page, then the gpu is a happy camper and a hard worker. However, if you have 101 things to draw, but the 101st thing is on a second texture page then the gpu gets grumpy. He has to stop using one texture and **swap** to the second texture page. This is the texture swap. If the gpu has to swap textures a bunch of times each frame then it can become a performance bottleneck. This includes surface operations.

Now the question is: how many is too many? In reality, the number is far greater than most people say it is, but is still limited by platform and hardware. An HTML5 game is going to have a low threshold for swaps, a mobile game will also have a low-end threshold. A decently modern computer? The threshold is huge. You will know how many is too many when the game is beginning to chug on your minimum-spec benchmark system.

The next question is: when to start optimizing texture swaps out of the game? Realistically you probably do not need to. There are steps you can take to do this, self texture atlasing and using texture groups is the big one. Using a single object to batch your draws can also work. But if you spend ten hours trying to reduce texture swaps from 20 to 19, then you wasted ten hours. This doesn't mean you are free to do whatever you want; you should still be cautious and aware. But you should not let this number keep you up at night.

> Texture color misconception!
>
> Textures are sent the gpu as uncompressed, 24bit images. This means the memory footprint of a texture is Width * Height * 4 bytes, regardless of whether you use only three colors for your Downwell clone or the entire gamut plus some colors you invented. Your palette choice should never be a performance choice, and always be a choice of art direction!

![image](/topics/debugoverlay/img/break.png)

## batch breaks

Batch breaks are very similar to texture swaps; this is the number of times the gpu state has to change. So similar, that a texture swap causes a batch break! Put more plainly, it is the number of times you yell at the gpu to do something different. Similarly to texture swaps, how high this number can go is often battled about across the world wide web and likewise the true limit isn't just some exact number. Just like swaps, you need to know what a batch break is.

Mentioned earlier, the gpu is very good at its job but it doesn't like being told what to do a godzillion times per frame. There are certain functions and operations that make the gpu halt its current task, change its state, and continue. That is a batch break. Literally, it has to stop working on the instructions for the last batch of drawing and now focus on a new batch. For example, pretend you need to draw some text, draw a sprite with a shader, then draw more text:
```gml
draw_text(x, y, "yo momma!");

shader_set(funny_clown_shader) {
	draw_sprite(spr_yourmom, 0, x, y);
	shader_reset();
}

draw_text(x + 20, y, "is a clown!");
```

Setting the shader like this is a change in gpu instruction and will cause a batch break. The number of things that cause this is not officially listed anywhere but we've got a good grip on it. Drawing certain primitives, setting shaders, blendmodes, texture filtering or repeating, submitting vertex buffers, matrix operations, and many more things. Breaking the batch is unavoidable, but like texture swaps, can be mitigated. By grouping your gpu instruction-changing calls together you can reduce the number of batch breaks to bring this number down. But the same advice as lowering swaps applies to lowering batch breaks: the upper limit on batch breaks is hardware and platform dependent and a high number is not automatically bad. Keep an eye on the number, and if you get game chug and are having trouble narrowing down the cause, then making steps to reduce batch breaks can help.

> Common batch break no-no!
>
> It is not uncommon to set the same shader for objects in their draw events one by one. Don't do this! Consider using the layer_script_being() and end() functions to batch your shader usage for a single shader_set() call.

![image](/topics/debugoverlay/img/graph.png)

## the graph

We've arrived at what is possibly the most confusing part of the debug overlay: the graph. The [manual](https://manual.yoyogames.com/GameMaker_Language/GML_Reference/Debugging/show_debug_overlay.htm) lists what each color of the graph represents but I would like to expand on it. This graph can be considered a "what thing is doing the most damage to me right now" graph. The two most common colors you will see on this graph are **red** and **yellow**. Red is your game's step event, the bigger the red bar the longer your game's step event is taking to resolve. Yellow is your game's draw event, and a big yellow bar means a big huge amount of time used for drawing. Keep in mind, that just because these bars are big does not mean your game is broken. A game with a lot of visual flare is going to have a big yellow bus and a Dwarf Fortress like game is going to have red lipstick smeared across the top of their game. That being said, if you see a lot of red and a lot of yellow, and your game is chug chug chuggin then it is time to run the game in debug mode, and start the profiler. From the profiler you will be able to see that calling `point_distance()` 12,000 times per step, or calling `draw_text()` for each individual character in a title screen is killing your game. But let's talk about blue.

**Blue** is the time the garbage collector (gc) spends picking up your mess. This is a bit more mysterious, because the ins and outs of the garbage collector are not written anywhere. The gc's job is to take anything allocated in memory that is no longer being used, and yeet it entirely. Thing is, the gc is not first-in-first-out, and it does not just deal with data structures. If you `instance_destroy()` something, the garbage collector has to deal with it. If you free a surface, delete a sprite, or destroy a particle emitter then the garbage man is tasked with deallocating the memory used for those things.

But the gc is can only do so much it is no superman. Not everything marked for deallocation is getting dealt with when it collects. It reacts to pressure, and it doesn't care what order you tell it to work in. It is also smart enough to not deallocate things you may reuse later like string related buffers. But the gc does aim to please, and the more you throw at it the harder it works. This will slow your game down considerably in some situations. Arrays and structs are very big garbage offenders. Unlike data structures, the runner handles freeing arrays and structs from memory for you so it is very tempting to create a brazillion of them every frame without a care in the world. No! Bad! These back the gc up, giving it more and more work to do each collection. If you are seeing a lot of blue, double check that vector library you just wrote and consider toning down all your `return new vec2()` calls. If you are not seeing a lot of blue, don't worry.

> Instance pools
>
> Creating and destroying instances frequently not only puts a strain on the garbage collector, the related functions have a lot of overhead. Consider doing some research into object pooling. For example if your shmup is destroying every single bug alien when it flies past the player; instead of destroying them send them back to the top of the level and reset their state machine so they can wait until its their time to shine again. This technique can be used for bullets as well!