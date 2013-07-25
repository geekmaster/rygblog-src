-title=A trip through the Graphics Pipeline 2011, part 6
-time=2011-07-06 11:22:29
*This post is part of the series ["A trip through the Graphics Pipeline 2011"](*a-trip-through-the-graphics-pipeline-2011-index).*

Welcome back. This time we're actually gonna see triangles being rasterized \- finally! But before we can rasterize triangles, we need to do triangle setup, and before I can discuss triangle setup, I need to explain what we're setting things up *for*; in other words, let's talk hardware\-friendly triangle rasterization algorithms.

### How *not* to render a triangle

First, a little heads\-up to people who've been at this game long enough to have written their own optimized software texture mappers: First, you're probably used to thinking of triangle rasterizers as this amalgamated blob that does a bunch of things at once: trace the triangle shape, interpolate u and v coordinates \(or, for perspective correct mapping, u/z, v/z and 1/z\), do the Z\-buffer test \(and for perspective correct mapping, you probably used a 1/z buffer instead\), and then do the actual texturing \(plus shading\), all in one big loop that's meticulously scheduled and probably uses all available registers. You know the kind of thing I'm talking about, right? Yeah, forget about that here. This is hardware. In hardware, you package things up into nice tidy little modules that are easy to design and test in isolation. In hardware, the "triangle rasterizer" is a block that tells you what \(sub\-\)pixels a triangle covers; in some cases, it'll also give you barycentric coordinates of those pixels inside the triangle. But that's it. No u's or v's \- not even 1/z's. And certainly no texturing and shading, through with the dedicated texture and shader units that should hardly come as a surprise.

Second, if you've written your own triangle mappers "back in the day", you probably used an incremental scanline rasterizer of the kind described in Chris Hecker's [series on Perspective Texture Mapping](http://chrishecker.com/Miscellaneous_Technical_Articles). That happens to be a great way to do it in sofware on processors without SIMD units, but it doesn't map well to modern processors with fast SIMD units, and even worse to hardware \- not that it's stopped people from trying. In particular, there's a certain dated game console standing in the corner trying very hard to look nonchalant right now. The one with that triangle rasterizer that had *really* fast guard\-band clipping on the bottom and right edges of the screen, and not so fast guard\-band clipping for the top and left edges \(that, my friends, is what we call a "tell"\). Just saying.

So, what's bad about that algorithm for hardware? First, it really rasterizes triangles scan\-line by scan\-line. For reasons that will become obvious once I get to Pixel Shading, we want our rasterizer to output in groups of 2x2 pixels \(so\-called "quads" \- not to be confused with the "quad" primitive that's been decomposed into a pair of triangles at this stage in the pipeline\). This is all kinds of awkward with the scan\-line algorithm because not only do we now need to run two "instances" of it in parallel, they also each start at the first pixel covered by the triangle in their respective scan lines, which may be pretty far apart and doesn't nicely lead to generating the 2x2 quads we'd like to get. It's also hard to parallelize efficiently, not symmetrical in the x and y directions \- which means a triangle that's 8 pixels wide and 100 pixels stresses very different parts of the rasterizer than a triangle that's 100 pixels wide and 8 pixels high. Really annoying because now you have to make the "x" and "y" stepping "loops" equally fast in order to avoid bottlenecks \- but we do all our work on the "y" steps, the loop in "x" is trivial! As said, it's a mess.

### A better way

A much simpler \(and more hardware\-friendly\) way to rasterize triangles was presented in a 1988 [paper](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.157.4621&rep=rep1&type=pdf) by Pineda. The general approach can be summarized in 2 sentences: the signed distance to a line can be computed with a 2D dot product \(plus an add\) \- just as a signed distance to a plane can be compute with a 3D dot product \(plus add\). And the interior of a triangle can be defined as the set of all points that are on the correct side of all three edges. So... just loop over all candidate pixels and test whether they're actually inside the triangle. That's it. That's the basic algorithm.

Note that when we move e.g. one pixel to the right, we add one to X and leave Y the same. Our edge equations have the form $$E(X,Y) = aX + bY + c$$, with a, b, c being per\-triangle constants, so for X\+1 it will be $$E(X+1,Y) = a(X+1) + bY + c = E(X,Y) + a$$. In other words, once you have the values of the edge equations at a given point, the values of the edge equations for adjacent pixels are just a few adds away. Also note that this is absolutely trivial to parallelize: say you want to rasterize 8x8 = 64 pixels at once, as AMD hardware likes to do \(or at least the Xbox 360 does, according to the 3rd edition of [Real-time Rendering](http://realtimerendering.com/book.html)\). Well, you just compute $$ia + jb$$ for $$0 \le i, j \le 7$$ once for each triangle \(and edge\) and keep that in registers; then, to rasterize a 8x8 block of pixels, you just compute the 3 edge equation for the top\-left corner, fire off 8x8 parallel adds of the constants we've just computed, and then test the resulting sign bits to see whether each of the 8x8 pixels is inside or outside that edge. Do that for 3 edges, and presto, one 8x8 block of a triangle rasterized in a truly embarrassingly parallel fashion, and with nothing more complicated than a bunch of integer adders! And by the way, this is why there's snapping to a fixed\-point grid in the previous part \- so we can use integer math here. Integer adders are much, *much* simpler than any floating\-point math unit. And of course we can choose the width of the adders just right to support the viewport sizes we want, with sufficient subpixel precision, and probably a 2x\-4x factor on top of that so we get a decently\-sized guard band.

By the way, there's another thorny bit here, which is fill rules; you need to have tie\-breaking rules to ensure that for any pair of triangles sharing an edge, no pixel near that edge will ever be skipped or rasterized twice. D3D and OpenGL both use the so\-called "top\-left" fill rule; the details are explained in the respective manuals. I won't talk about it here except to note that with this kind of integer rasterizer, it boils down to subtracting 1 from the constant term on some edges during triangle setup. That makes it guaranteed watertight, no fuss at all \- compare with the kind of contortions Chris has to go through in his article to make this work properly! Sometimes things just come together beautifully.

We have a problem though: How do we find out which 8x8 blocks of pixels to test against? Pineda mentions two strategies: 1\) just scanning over the whole bounding box of the triangle, or 2\) a smarter scheme that stops to "turn around" once it notices that it didn't hit any triangle samples anymore. Well, that's just fine if you're testing one pixel at a time. But we're doing 8x8 pixels now! Doing 64 parallel adds only to find out at the very end that exactly none of them hit any pixels whatsoever is a *lot* of wasted work. So... don't do that!

### What we need around here is more hierarchy

What I've just described is what the "fine" rasterizer does \(the one that actually outputs sample coverage\). Now, to avoid wasted work at the pixel level, what we do is add another rasterizer in front of it that doesn't rasterize the triangle into pixels, but "tiles" \- our 8x8 blocks \([This](http://people.csail.mit.edu/ericchan/bib/pdf/p15-mccormack.pdf) paper by McCormack and McNamara has some details, as does Greene's ["Hierarchical Polygon Tiling with Coverage Masks"](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.115.1646&rep=rep1&type=pdf) that takes the idea to its logical conclusion\). Rasterizing edge equations into covered tiles works very similarly to rasterizing pixels; what we do is compute lower and upper bounds for the edge equations over full tiles; since the edge equations are linear, such extrema occur on the boundary of the tile \- in fact, it's enough to loop at the 4 corner points, and from the signs of the 'a' and 'b' terms in the edge equation, we can determine which corner. Bottom line, it's really not much more expensive than what we already discussed, and needs exactly the same machinery \- a few parallel integer adders. As a bonus, if we evaluate the edge equations at one corner of the tile anyway, we might as well just pass that through to the fine rasterizer: it needs one reference value per 8x8 block, remember? Very nice.

So what we do now is run a "coarse" rasterizer first that tells us which tiles might be covered by the triangle. This rasterizer can be made smaller \(8x8 at this level really seems like overkill!\), and it doesn't need to be as fast \(because it's only run for each 8x8 block\). In other words, at this level, the cost of discovering empty blocks is correspondingly lower.

We can think this idea further, as in Greene's paper or Mike Abrash's description of [Rasterization on Larrabee](http://drdobbs.com/architecture-and-design/217200602), and do a full hierarchical rasterizer. But with a hardware rasterizer, there's little to no point: it actually *increases* the amount of work done for small triangles \(unless you can skip levels of the hierarchy, but that's not how you design HW dataflows!\), and if you have a triangle that's large enough to actually produce significant rasterization work, the architecture I describe should already be fast enough to generate pixel locations faster than the shader units can consume them.

In fact, the actual problem here isn't big triangles in the first place; they are easy to deal with efficiently for pretty much any algorithm \(certainly including scan\-line rasterizers\). The problem is small triangles! Even if you have a bunch of tiny triangles that generate 0 or 1 visible pixels, you still need to go through triangle setup \(that I *still* haven't described, but we're getting close\), at least one step of coarse rasterization, and then at least one fine rasterization step for an 8x8 block. With tiny triangles, it's easy to get either triangle setup or coarse rasterization bound.

One thing to note is that with this kind of algorithm, slivers \(long, very thin triangles\) are seriously bad news \- you need to traverse tons of tiles and only get very few covered pixels for each of them. So, well, they're slow. Avoid them when you can.

### So what does triangle setup do?

Well, now that I've described what the rasterization algorithm is, we just need to look what per\-edge constants we used throughout; that's exactly what we need to set up during triangle setup.

In our case, the list is this:

* The edge equations \- a, b, c for all 3 triangle edges.
* Some of the derived values, like the $$ia + jb$$ for $$0 \le i, j \le 7$$ that I mentioned; note that you wouldn't actually store a full 8x8 matrix of these in hardware, certainly not if you're gonna add another value to it anyway. The best way to do this is in HW probably to just compute the $$ia$$ and $$jb$$, use a [Carry-save adder](http://en.wikipedia.org/wiki/Carry-save_adder) \(aka 3:2 reducer, I wrote about them [before](*carry-save-adders-and-averaging-bit-packed-values)\) to reduce the $$ia + jb + c$$ expression to a single sum, and then finish that off with a regular adder. Or something similar, anyway.
* Which reference corner of the tiles to use to get the upper/lower bounds of the edge equations for coarse rasterizer.
* The initial value of the edge equations at the first reference point for the coarse rasterizer \(adjusted for fill rule\).

...so that's what triangle setup computes. It boils down to several large integer multiplies for the edge equations and their initial values, a few smaller multiplies for the step values, and some cheap combinatorial logic for the rest.

### Other rasterization issues and pixel output

One thing I didn't mention so far is the scissor rect. That's just a screen\-aligned rectangle that masks pixels; no pixel outside that rect will be generated by the rasterizer. This is fairly easy to implement \- the coarse rasterizer can just reject tiles that don't overlap the scissor rect outright, and the fine rasterizer ANDs all generated coverage masks with the "rasterized" scissor rectangle \(where "rasterization" here boils down to a one integer compare per row and column and some bitwise ANDs\). Simple stuff, moving on.

Another issue is multisample antialiasing. What changes is now you have to test more samples per pixel \- as of DX11, HW needs to support at least 8x MSAA. Note that the sample locations inside each pixel aren't on a regular grid \(which is badly behaved for near\-horizontal or near\-vertical edges\), but dispersed to give good results across a wide range of multiple edge orientations. These irregular sample locations are a total pain to deal with in a scanline rasterizer \(another reason not to use them!\) but very easy to support in a Pineda\-style algorithm: it boils down to computing a few more per\-edge offsets in triangle setup and multiple additions/sign tests per pixel instead of just one.

For, say 4x MSAA, you can do two things in an 8x8 rasterizer: you can treat each sample as a distinct "pixel", which means your effective tile size is now 4x4 actual screen pixels after the MSAA resolve and each block of 2x2 locations in the fine rast grid now corresponds to one pixel after resolve, or you can stick with 8x8 actual pixels and just run through it four times. 8x8 seems a bit large to me, so I'm assuming that AMD does the former. Other MSAA levels work analogously.

Anyway, we now have a fine rasterizer that gives us locations of 8x8 blocks plus a coverage mask in each block. Great, but it's just half of the story \- current hardware also does early Z and hierarchical Z testing \(if possible\) before running pixel shaders, and the Z processing is interwoven with actual rasterization. But for didactic reasons it seemed better to split this up; so in the next part, I'll be talking about the various types of Z processing, Z compression, and some more triangle setup \- so far we've just covered setup for rasterization, but there's also various interpolated quantities we want for Z and pixel shading, and they need to be set up too! Until then.

### Caveats

I've linked to a few rasterization algorithms that I think are representative of various approaches \(they also happen to be all on the Web\). There's a lot more. I didn't even try to give you a comprehensive introduction into the subject here; that would be a \(lengthy!\) serious of posts on its own \- and rather dull after a fashion, I fear.

Another implicit assumption in this article \(I've stated this multiple times, but this is one of the places to remind you\) is that we're on high\-end PC hardware; a lot of parts, particularly in the mobile/embedded range, are so\-called *tile renderers*, which partition the screen into tiles and render each of them individually. These are *not* the same as the 8x8 tiles for rasterization I used throughout this article. Tiled renderes need at least another "ultra\-coarse" rasterization stage that runs early and finds out which of the \(large\) tiles are covered by each triangle; this stage is usually called "binning". Tiled renderers work differently and have different design parameters than the "sort\-last" architectures \(that's the official name\) I describe here. When I'm done with the D3D11 pipeline \(and that's still a ways off!\) I might throw in a post or two on tiled renderers \(if there's interest\), but right now I'm just ignoring them, so be advised that e.g. the PowerVR chips you so often find in smartphones handle some of this differently.

The 8x8 blocking \(other block sizes have the same problem\) means that triangles smaller than a certain size, or with inconvenient aspect ratios, take a lot more rasterization work than you would think, and get crappy utilization during the process. I'd love to be able to tell you that there's a magic algorithm that's easy to parallelize *and* good with slivers and the like, but if there is I don't know it, and since there's still regular reminders by the HW vendors that slivers are bad, apparently neither do they. So for the time being, this just seems to be a fact of life with HW rasterization. Maybe someone will come up with a great solution for this eventually.

The "edge function lower bound" thing I described for coarse rast works fine, but generates false positives in certain cases \(false positives in the sense that it asks for fine rasterization in blocks that don't actually cover any pixels\). There's tricks to reduce this, but again, detecting some of the rarer cases is trickier / more expensive than just rasterizing the occasional fine block that doesn't have any pixels lit. Another trade\-off.

Finally the blocks used during rasterization are often snapped on a grid \(why that would help will become clearer in the next part\). If that's the case, even a triangle that just covers 2 pixels might straddle 2 tiles and make you rasterize two 8x8 blocks. More inefficiency.

The point is this: Yes, all this is fairly simple and elegant, but it's not perfect, and actual rasterization for actual triangles is nowhere near theoretical peak rasterization rates \(which always assume that all of the fine blocks are completely filled\). Keep that in mind.