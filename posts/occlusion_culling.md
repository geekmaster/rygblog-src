-time=2013-02-17
# Optimizing Software Occlusion Culling

In January of 2013, some nice folks at Intel released a [Software Occlusion
Culling demo](http://software.intel.com/en-us/vcsource/samples/software-occlusion-culling)
with full source code. I spent about two weekends playing around with the code,
and after realizing that it made a great example for various things I'd been
meaning to write about for a long time, started churning out blog posts about
it for the next few weeks. This is the resulting series.

Here's the list of posts (the series is now finished):

1. "[%](*write_combining)", on typical write combining issues when writing graphics code.
2. "[%](*string_processing_rant)", a slightly over-the-top post that starts with some bad string processing habits and ends in a rant about what a complete minefield the standard C/C++ string processing functions and classes are whenever non-ASCII character sets are involved.
3. "[%](*cores_dont_share)", on some very common pitfalls when running multiple threads that share memory.
4. "[%](*fixing_cache_lazy)". You could redesign your system to be more cache-friendly - but when you don't have the time or the energy, you could also just do this.
5. "[%](*frustum_culling_turning_crank)" --- on the other hand, if you do have the time and energy, might as well do it properly.
6. "[%](*barycentric_conspiracy)" is a lead-in to some in-depth posts on the triangle rasterizer that's at the heart of Intel's demo. It's also a gripping tale of triangles, Möbius, and a plot centuries in the making.
7. "[%](*tri_rast_in_practice)" --- how to build your own precise triangle rasterizer and *not* die trying.
8. "[%](*optimizing_basic_rasterizer)", because this is real time, not amateur hour.
9. <a href="http://fgiesen.wordpress.com/2013/02/11/depth-buffers-done-quick-part/">"Depth buffers done quick, part 1"</a> - at last, looking at (and optimizing) the depth buffer rasterizer in Intel's example.
10. <a href="http://fgiesen.wordpress.com/2013/02/16/depth-buffers-done-quick-part-2/">"Depth buffers done quick, part 2"</a> - optimizing some more!
11. <a href="http://fgiesen.wordpress.com/2013/02/17/care-and-feeding-of-worker-threads-part-1/">"The care and feeding of worker threads, part 1"</a> - this project uses multi-threading; time to look into what these threads are actually doing.
12. <a href="http://fgiesen.wordpress.com/2013/02/25/the-care-and-feeding-of-worker-threads-part-2/">"The care and feeding of worker threads, part 2"</a> - more on scheduling.
13. <a href="http://fgiesen.wordpress.com/2013/02/28/reshaping-dataflows/">"Reshaping dataflows"</a> - using global knowledge to perform local code improvements.
14. <a href="http://fgiesen.wordpress.com/2013/03/04/speculatively-speaking/">"Speculatively speaking"</a> - on store forwarding and speculative execution, using the triangle binner as an example.
15. <a href="http://fgiesen.wordpress.com/2013/03/05/mopping-up/">"Mopping up"</a> - a bunch of things that didn't fit anywhere else.
16. "[The Reckoning](*occlusion_reckoning)" --- in which a lesson is learned, but [the damage is irreversible](http://www.alessonislearned.com/).

All the code is available on [Github](https://github.com/rygorous/intel_occlusion_cull/); there's
various branches corresponding to various (simultaneous) tracks of development,
including a lot of experiments that didn't pan out. The articles all reference
the [blog branch](https://github.com/rygorous/intel_occlusion_cull/tree/blog)
which contains only the changes I talk about in the posts - i.e. the
stuff I judged to be actually useful.

Special thanks to Doug McNabb and Charu Chandrasekaran at Intel for publishing
the example with full source code and a permissive license, and for saying
"yes" when I asked them whether they were okay with me writing about my
findings in this way!

<a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/">
<img src="http://i.creativecommons.org/p/zero/1.0/88x31.png" style="border-style:none;" alt="CC0">
</a>
<br>
To the extent possible under law,
<a rel="dct:publisher" href="http://blog.rygorous.org">
<span>Fabian Giesen</span></a>
has waived all copyright and related or neighboring rights to
<span>Optimizing Software Occlusion Culling</span>.
