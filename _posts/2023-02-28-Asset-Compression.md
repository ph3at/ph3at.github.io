---
title: Game Asset Storage, Loading, Compression and Caching
date: 2023-02-28 19:00:00 +0100
categories: [Programming, Game Development]
tags: [programming, compression, archives, loading, caching, directstorage]
author: petert
img_path: /assets/2023-02-28-Asset-Compression
---

# Background and DirectStorage

Compression, archiving and loading of game assets has been an important topic in game development for several decades,
but it relatively recently gained some mainstream interest due to the marketing associated with the latest high-end
consoles to be released. On PC, equivalent features are now provided using the 
[DirectStorage API](https://devblogs.microsoft.com/directx/directstorage-1-1-now-available/), which includes
both higher-performance primitives for basic file access as well as GPU-accelerated decompression.

However, this blog post is not about that. Rather, it is about **general concepts of asset storage and compression**,
and the various factors that play into decisions regarding that. As is not too uncommon for PH3, we'll focus in
particular on a few **aspects that primarily affect PC** and are frequently overlooked -- above all, actually
utilizing the frequently larger amounts of main memory available compared to other platforms.

While DirectStorage and GPU decompression are interesting, their impact specifically on PC is often vastly
overestimated in common discourse. PC CPUs are *really fast*, and so are PCIe links. The difference between good
and bad general software engineering practices in asset storage formats, compression, access and caching in load times
is far greater than what additional APIs or hardware acceleration provide. Of course, when everything else
is already highly optimized and following best practices, *then* going with hardware acceleration and specialized
APIs can give you an extra boost.  
But the vast majority of games are absolutely nowhere close to e.g. saturating
the 32 GB/s bandwidth provided by PCIe 4.0 -- doing so would mean that a loading time of 500 ms (half a second) would be
enough to entirely replace **all** the main memory content of a 16 GB GPU -- never mind that much of that will be taken
up by e.g. framebuffer data which never traverses the bus.

> ComputerBase recently [analyzed the impact of DirectStorage on the load times in Forspoken](https://www.computerbase.de/2023-01/forspoken-directstorage-benchmark-test/#abschnitt_directstorage_mit_nvme_und_sata_im_test).
> In the longest-loading scene, on the fastest hardware in the test, having DirectStorage enabled resulted in load times of 2.0 seconds, while disabling it increased that to 2.3. 
> A difference of 15% isn't completely negligible, but with many games loading for 10s of seconds on similar hardware, there is clearly low-hanging engineering fruit which needs to be exploited before
> going with specialized APIs or hardware decompression will truly yield benefits. 
{: .prompt-info }

# Hot and Cold Loading

When discussing asset loading performance, particularly on Windows PCs, it's essential to distinguish between "Hot"
and "Cold" load scenarios. **As long as some RAM is available, the OS will cache files in memory and serve I/O requests from there.**
Understanding this is fundamental when trying to analyze and optimize loading performance, as it will generally
shift the bottleneck from I/O to the CPU.

> During development, a relatively convenient way to create (and measure) a cold loading scenario is by using the 
> [RAMMap tool from the SysInternals suite](https://learn.microsoft.com/en-us/sysinternals/downloads/rammap). 
> It allows you to clear the OS standby file set, which means that subsequent loads will be "cold".
{: .prompt-tip }

I talked about this 
[a while back](https://steamcommunity.com/games/991270/announcements/detail/1806446683762590792) 
when I introduced our initial attempt at creating an asset compression and packaging toolchain for Trails of Cold Steel 3,
which was quite successful and which we also re-used for the fourth game in the series. 
Here is the chart I produced back then, showing the cold and hot loading times as well as storage savings of various
compression and packaging schemes.  

![Hot and cold load times with various compression schemes in Trails of Cold Steel 3](hot_cold_load_tocs3.png){: #fancyimg }
_Hot and cold load times with various compression schemes in Trails of Cold Steel 3_

Back then we decided to settle on using LZ4 instead of ZSTD. When comparing both in a PKA container (with deduplication), 
the latter is substantially smaller, but does also increases loading times in a "hot load" scenario substantially.

While 4 GB or so are not the end of the world, one thing that has changed is that now, the **Steam Deck** exists, 
is a great option to play many of the games we port, and has a very attractively priced **64 GB** version available. 
I always felt like it would be great to be able to have your cake and eat it too when it comes
to this file size and hot loading speed tradeoff, and with the Deck it's even more critical. 
But before we get to that (spoiler: it's mostly possible to have both) -- let's quickly talk about compression.

# Compression Algorithms

Both general purpose compression and compression algorithms for specific types of data are still an active research area,
with the trade-off between compression ratio and throughput being of essential importance.

> While this is outside the games context, a colleague of mine and I relatively recently published 
> [a compression algorithm specifically designed for scientific data in high performance computing](https://ieeexplore.ieee.org/document/9910073).
{: .prompt-info }

However, for the use case of general data (as opposed to e.g. textures) compression, where the focus is on high decompression
throughput, there are two algorithms -- and crucially, industrial-strength implementations -- that stick out:
[LZ4](https://lz4.github.io/lz4/) and [ZStandard](https://facebook.github.io/zstd/).

 * **LZ4** has been around longer, is extremely fast, and has moderate tuning potential regarding the tradeoff between compression
speed and ratio.  
 * **ZStandard** can achieve even higher compression rates, at the cost of somewhat slower -- but still extremely
fast, in the grand scheme of things -- decompression speeds. 

Both of them are extremely well implemented and have great documentation, and neither is very difficult to get up and running
from an implementation perspective.  
Note that in the game asset compression scenario we don't really
care too deeply about the *compression* speed part of the equation -- that happens once when the build is made, and as long as 
it doesn't take hours everything else is no big deal. What we do care about is decompression, and as shown in the ToCS3 chart,
if we expect a lot of hot loading there is a non-negligible performance loss involved with choosing Zstandard over LZ4.

## An Aside: Zstandard Dictionaries

One exciting feature of Zstandard is the possibility to use [dictionary compression](https://facebook.github.io/zstd/#small-data).
In basic terms, a dictionary is trained for a specific type of data, and this dictionary can then be re-used in order to
compress and decompress files of that type more effectively. Dictionary compression can improve both the compression ratio and
loading speed, especially if the dictionary is kept in memory in a prepared form.

Sadly, for many game asset storage purposes, dictionaries are not particularly useful. They excel when individual files are **very
small** (i.e. in the kB range and below), and there are a great many of them. With increasing file sizes, the advantages compared
to non-dictionary ZSTD mostly evaporate. 

However, there is one type of asset in games which fits this bill perfectly: **shader files!**  
The following table shows the results -- in terms of decompression speed and resulting total file size -- for the set of shaders 
in one of our ports, which in uncompressed form are `7104` files with a total size on disk of `76 MB`.

[comment]: # Amazing misuse of &nbsp; to make table column spacing more equal

|              | &nbsp; &nbsp;LZ4 | &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ZSTD   | dict ZSTD |
|--------------|-------:|-------:|----------:|
| Speed (MB/s) |  589.3 |  351.5 |     610.3 |
| Size (MB)    |   22.8 |   16.3 |       5.7 |
{: style="margin: auto;"}

ZSTD with a dictionary outperforms LZ4 (and uncompressed loading!) in speed, while being less than 1/4th its size.
This is an absolutely amazing result, and while obviously the total size of these shaders is too small for any of this to truly matter,
it nonetheless motivated me to include and use ZSTD dictionary support in our new general purpose asset storage format.

# Caching Everything

Clearly, we'd like to use Zstandard for (almost) everything, but there is that pesky hot loading time impact. While
with properly optimized loading it is less than a second in our use cases, that's still roughly a second of extra wait time
which happens every time a level is loaded, which in a long JRPG can be quite a few times. That's something which irks me 
personally, and luckily, there is a solution.

Many modern PCs have 16, 32 or in some cases even 64 GB of main memory -- in fact, according to the [latest Steam Hardware Survey](https://store.steampowered.com/hwsurvey), 
more than 51% of all PCs used with Steam now have 16 GB or more. When someone is playing almost any game, the vast majority
of that memory sits unused, especially in the 32 or 64 GB case.

Obviously, it's not advisable to *require* 32 GB or, worse,
64 GB, so we cannot use it for strictly necessary data. However, nothing prevents a PC game from querying the amount of
available memory -- using, for example, the `GlobalMemoryStatusEx` [function on Windows](https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-globalmemorystatusex)
 -- and using a good portion of it for caching.

> When looking at the `MEMORYSTATUSEX` structure in Windows, it's tempting to believe that an application can allocate
> `ullAvailPhys` bytes of memory -- or at least close to that. In our experience, this can fail depending on the virtual
> memory setup of a given PC. What we found to be safe is to use up to `min(ullAvailPhys, ullAvailPageFile) * 0.8`.
> I am still unsure why exactly, in Windows, free physical memory sometimes cannot be used because of a lack of page file space. 
{: .prompt-warning }

# Putting it all Together -- the P3A Format

Taking all of these facts into account, we designed a new asset delivery format for our future games.
In a show of creative genius, we decided to call it **P3A**, or the **P**H**3** **A**rchive format.

Implementation-wise, this includes two main components:

 * **P3ATool**, a program with the primary purpose of building P3A files. It can also extract their contents,
   check file integrity, and provide meta-information about the data contained.
 * The P3A **Archivist**, a library component which is intended to load the contents of P3A files in
   games. And this is the component which also supports the **adaptive caching** discussed previously.

In its current version, a P3A file may contain any number of individual asset files, each of which can
be either uncompressed, LZ4 compressed, ZSTD compressed, or using ZSTD compression with a custom dictionary.
The header contains a checksum for each file, as we've seen a larger number of strange and unique bug
reports due to filesystem corruption than one would expect.
There is also a configurable alignment for files, both for access efficiency and to enable smaller
incremental patches in some circumstances.

> One niche but relatively frequent issue with proprietary game packaging formats is that they make it 
> harder for modders to access game data. In order to prevent that issue, we decided to ship `p3atool.exe` 
> with every game that uses this format.
{: .prompt-info }

For cases where large files would need to be added post-release, the Archivist component also supports
patching by overriding the contents of previous archives with an entirely new one, again enabling
smaller patch downloads with little extra work and no impact on game loading performance. 

# Some Results

We have implemented this file format for the first time in a soon-to-be-shipped game, and the results
are very encouraging. The following chart shows a comparison between the best option without any packaging and while
using automatic OS caching (i.e. the Windows file cache, blue), the best P3A packaged option without caching (red), 
and finally the shipping implementation using P3A and Archivist caching (green).

![Loading times with various packaging and caching options, across 3 game areas](cached_p3a_loading.png){: #fancyimg }
_Loading times with various packaging and caching options, across 3 game areas_

Note that compressed packaging without full caching (red) results in slightly longer
average hot load times than using uncompressed data and letting the operating system take care of caching
all accesses (blue). This is due to the overhead involved in decompression -- as discussed previously,
this overhead is small (between 0.1 and 0.4 seconds in this experiment), but it does exist. 

When using **full P3A caching** -- assuming sufficient memory is available, of course -- *loading times drop
substantially below even the uncompressed and OS cached results*. And not just that, since no I/O is involved
at this point, it also increases the performance consistency and reduces the potential for frame time
fluctuations introduced by storage access operations.

# Conclusion & Outlook

So, to summarize:

 * Asset compression is obviously useful to reduce storage and download sizes, but can also reduce cold
   load times depending on the relative speed of the I/O subsystem and the decompression rate.
 * With high-throughput compression algorithms such as LZ4 and ZStandard, even CPU decompression is
   very fast on current CPUs.
 * Any remaining minor hot load time increases can be mitigated by in-memory caching of decompressed assets
   (as long as sufficient unused memory is available) -- and this can even further improve performance
   and frametime consistency.
 * On PC specifically, it might be a good idea to dynamically adjust caching behavior to the hardware and
   software stack on a given system.

Of course it is very likely that, at least for data that eventually ends up on the GPU, using DirectStorage
with GPU decompression could improve load times even further. However, there isn't that much room to go
down from 500-800ms in the first place, and since most of that time isn't spent in decompression it's likely 
that any further improvement might be in the single-digit percentages.

Nonetheless, the P3A format is sufficiently flexible to accommodate arbitrary compression algorithms, so
adding e.g. support for the GPU-accelerated `GDeflate` in the future is certainly an option.
