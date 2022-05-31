![image](/topics/debugoverlay/img/header.png)

# the debug overlay _!_

## [ scary graph noises intensify ]

So you found `show_debug_overlay()` in the manual, and now you're looking at it and it's moving so fast and all the colors and numbers are confusing and you just want to know why your game is lagging.

I have some bad news. The debug overlay isn't going to be able to tell you why your game is broken, especially if it is broken for shader reasons. It can however, give you a good _idea_ of where to start looking to figure out what broke the game.

## the gist

The debug overlay is a graph. It has the following puzzle pieces: your `fps_real`, the amount of time specific processes are taking to resolve, the number of texture swaps your game has, and the number of batch breaks. This data gives you a decently accurate picture of your game's performance state and is best used to check in on occasionally.

![image](/topics/debugoverlay/img/fpsreal.png)

## fps_real

This is perhaps the most misleading thing you can look at to judge your game's performance. First of all, this is a measure of how fast your game could run if it were not bottlenecked by cpu cycle times. To make a complex topic into a thin soup: your game is a constant loop and each iteration of the loop finishes before the cpu cycle does. In order to conserve power, your game sleeps for the remainder of that cycle then wakes up to start the next iteration. If your cpu did not have cycles, and your game did not go to sleep each cycle, this is (sort of) how many frames per second your game would run. Adding or removing things from your game can alter this number misleadingly, and it can also be different based on how many youtube videos you're watching at once. This is an unrealistic thing to measure your game by, and you should instead just babysit this number until it gets to an uncomfortable point.

Personally I treat fps_real like a budget for cool stuff in my game, and just keep adding cool stuff until fps_real gets in between two and five hundred.

![image](/topics/debugoverlay/img/swap.png)

## texture swaps

Texture swaps and batch breaks are things you will see various folks on the internet screaming about keeping low, which can make these numbers scary as they get high. In reality its not as simple as **low number good big number bad ooga ooga**. But! Before you can understand how big this number can be before it's too big, you need to understand what a texture swap is.

When you're fiddlin' about in gm and you add a sprite to your game it gets put on something called a texture page. This is like a big sticker sheet of your game's sprites. Each page can only hold so many sprites, so when you add more and more sprites you end up with multiple texture pages. When your game goes to draw something, the cpu passes the texture page to the gpu, along with information about what on the texture page needs to be drawn.

Well the gpu is very good at its job. If you have a hundred sprites to be drawn, and they're all on the same texture page, then the gpu is a happy camper and a hard worker. However, if you have 101 things to draw, but the 101st thing is on a second texture page then the gpu gets grumpy. He has to stop using one texture and **swap** to the second texture page. This is the texture swap. If the gpu has to swap textures a bunch of times each frame then it can become a performance bottleneck.

Now the question is: how many is too many? In reality, the number is far greater than most people say it is, but is still limited by platform and hardware. An HTML5 game is going to have a low threshold for swaps, a mobile game will also have a low-end threshold. A decently modern computer? The threshold is huge. You will know how many is too many when the game is beginning to chug on your minimum-spec benchmark system.

The next question is: when to start optimizing texture swaps out of the game? Realistically you probably do not need to. There are steps you can take to do this, self texture atlasing and using texture groups is the big one. Using a single object to batch your draws can also work. But if you spend ten hours trying to reduce texture swaps from 20 to 19, then you wasted ten hours. This doesn't mean you are free to do whatever you want; you should still be cautious and aware. But you should not let this number keep you up at night.

![image](/topics/debugoverlay/img/break.png)

## batch breaks

Batch breaks are very similar to texture swaps; this is the number of times the gpu state has to change, or put more plainly, the number of times you yell at the gpu to do something different. Similarly to texture swaps, how high this number can go is often battled about across the world wide web and likewise the true limit isn't just some exact number. Just like understanding the swap limit, you need to know what a batch break is.

Mentioned ealier, the gpu is very good at its job but it doesn't like being told what to do a godzillion times per frame. There are certain functions and operations that make the gpu halt its current task, change its state, and continue. That is a batch break. Literally, it has to stop working on the instructions for the last batch of drawing and now focus on a new batch. For example, pretend you need to make 10 `draw_text()` calls each frame with an alpha of .5 and then 10 more `draw_text()` calls with an alpha of 1. Your code might look like this:
```js
draw_set_alpha(0.5);
repeat (10) {
    draw_text(x, y, "yo momma!");
}
draw_set_alpha(1.0);
repeat (10) {}
	draw_text(x + 1, y - 1, "yo momma!");
}
```

This changes the global alpha state of your game, which is a change in gpu instruction and will cause a batch break. The number of things that cause this is pretty wild. Drawing certain primitives, changing alpha or font, submitting vertex buffers, surface targeting, and many more things. Breaking the batch is unavoidable, but like texture swaps can be mitigated. By grouping your gpu instruction-changing calls together you can reduce the number of batch breaks to bring this number down. But the same advice as lowering swaps applies to lowering batch breaks: the upper limit on batch breaks is hardware and platform dependent and a high number is not automatically bad. Keep an eye on the number, and if you get game chug and are having trouble narrowing down the cause then making steps to reduce batch breaks can help.