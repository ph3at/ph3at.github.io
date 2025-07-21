---
title: Evaluating VS Code LLM Agents on a Simple Web Site Update Task
date: 2025-07-23 19:30:00 +0100
categories: [LLMs, AI]
tags: [programming, html, llm, agent, vscode, evaluation]
author: petert
img_path: /assets/2025-07-23-LLM-Agents
---

# Background

This post (which is written entirely manually, in case you were curious) is quite different from what you generally
expect from our blog, in several ways. Usually, we talk about things like optimization, performance, or in-depth C++
programming topics -- all things that we are intimately familiar with, and that many people in PH3 have a decade or two
of professional experience with.

Today, we're talking about something completely different, and also something that I am personally not an expert in.
I have read quite a few papers, watched a few videos, and installed and evaluated a few local LLMs in the past, as well as
planned and supervised a student LLM hackathon project ([Hackaprompt 2024](https://sites.google.com/unitn.it/hackaprompt-2024/home)).
Which is to say, I know the principles and application, but I don't have nearly as much experience with it as I do with
the usual topics we cover here.

I've also personally tried various LLM coding assistants as a user over the past few years, in a decent variety of
languages and environments. To very shortly qualitatively summarize my experience, it's like this:

 * **C++ HPC development**: Worse than useless. For non-trivial things, you get code that is inefficient at best,
   plainly broken much of the time, and *subtly* broken at worst.
 * **C++ game development**: Probably a wash, on average. It can help with a few things, but may also introduce subtle
   bugs, and the time spent on fixing those can easily exceed the time saved. I'd say this also depends on the specific
   part of the codebase you are working in, with some parts being more amenable to LLM assistance than others.
 * **Jupyter notebooks**: Pretty good, actually. I am not at all an expert, and I've still run into things that the LLMs
   fail at completely even after repeated prompting (that I can figure out with a few minutes of googling and thinking),
   but this is something where I believe LLMs actually are a net positive for my personal productivity. I think it helps
   that these are usually smaller, self-contained, one-off tasks.
 * **C# Windows development**: Also not bad. While there can be some hallucinations, the end result might still be
   faster than navigating the API docs and writing the code yourself. Of course, if I did this as much as I do the C++
   tasks above, this might change, but for now, I find it useful.
 * **HTML/CSS**: This is where I find LLMs to be the most useful. Again, hardly my core area of expertise, but I can
   usually get a working result quickly. Again, I think it helps that these are just, well, overall "simpler" things, at
   least at the level I work with them (I can absolutely see that this might not be the case for enterprise-scale
   development).

> Quick aside -- *"What about unit tests?"*  
> I feel like there is a common idea that LLMs are better at writing unit tests than production code. In my experience,
> mostly with C++ in HPC, that is **not** the case. I think that, instead, this idea stems from the fact that some
> developers have *significantly lower standards for unit tests* than for production code. So if an LLM spits out
> something considered "good enough" for a unit test, it might not actually be better code -- just the same level of
> repetitive, DRY-violating mess which tests 500 cases that all go down the same code path, while missing another one
> that would actually be meaningful, but judged less harshly.  
> Then again, in our larger projects we have domain-specific, unit-tested unit testing helper infrastructure, so I might
> be biased here.
{: .prompt-tip }

As you can see, I'm not too much of a fan of LLMs overall, but I'm always interested in actually testing out any technology
in practice, and at least attempting to do so with some amount of rigor. Which brings us to this post, but before we
get to the experiment, there's one more bit of background that I need to provide.

## The PH3 Games Website

As probably not a lot of people know, [PH3 Games actually has a dedicated website](https://games.ph3.at/),
which is completely separate from our main website (which is also in dire need of updates or a redesign, but that's a
different topic).

This PH3 Games website is built as a static, manual, single HTML/CSS page, with no Javascript, no backend, and no
frameworks, so perhaps the simplest possible website you can imagine. I built it back when PH3 got started, and these
days I'd probably prefer some sort of Jekyll setup, but it's not a huge deal to maintain. It only serves to showcase our
projects, and contains a few quotes from our partners and reviews.

![An example gamecard from games.ph3.at](gamecard_example.png){: width="50%" #fancyimg }
_An example of a folded out gamecard on the PH3 games website_

Most relevant for this particular post, it contains a "gamecard" for each game project, with basic information about the
game, as well as a 2-3 paragraph text blurb. These blurbs were all written by me, and they don't focus on the game
itself -- beyond maybe a sentence to introduce context -- but rather on technical aspects, and how the project fits
into our portfolio and experience.

Now, if you know me, you know that I enjoy writing about our projects and their technical aspects. I might in fact do so
more than almost anyone else in a similar position, and you can find evidence of this in our 
[external links section](/external-posts/), as well as this blog itself.
However, these particular blurbs are mostly just a summary and aggregation of things I *have already* written about at
greater length, and writing them *again* in condensed form doesn't really add much value. So, I thought it would
be a good idea to try out LLMs for this task, and see how well they can do.

# The Experiment Setup

This evaluation was performed using the built-in VSCode LLM integration, using the models available to me at the time
of writing this post.

> Originally, I only planned to use the most recent version of each model family, but this part of the approach changed
> during the experiment, for reasons that will become clear later.
{: .prompt-info }

Of course, all models used the same prompt, which was as follows:

```text
Add a new gamecard (as the first in the list) for the Trails through Daybreak II PC port, released in 2025
Include these links:
https://store.steampowered.com/news/app/2668430/view/545600503107552212 (PC port feature overview)
https://store.steampowered.com/news/app/2668430/view/544477139480609941 (TAA and DLSS announcement)
The context for these is available in tmp. 
Carefully read that, as well as the existing game cards for "Trails through daybreak" and "Ys X", and match that style.
```

The `tmp` directory contained two HTML files which were dumped from their respective links. The other piece of context
provided was the `index.html` file, containing all existing gamecards, including those specifically mentioned in the
prompt.

> Initially, I did not bother with the local `tmp` directory for context and instead just told the LLM to read the
> websites. This produced no actual information, so I assume something with fetching the article from Steam failed
> (perhaps related to the content being delay-loaded somehow). I did not investigate further, the HTML file method
> worked, though it is more cumbersome. As this limitation affects all models equally, I do not consider it a problem
> for this evaluation.
{: .prompt-warning }

### Limitations

Note that this is not necessarily a reflection of the models' overall capabilities, but rather

{::nomarkdown}
<ul class="roman-list">
 <li> their performance on this specific task, 
 <li> as integrated within VS Code (integration which might differ in quality and capabilities between models), 
 <li> with the context provided by me and my skill (or lack thereof) at prompting, and 
 <li> in a not-statistically-evaluated 1-shot test.
</ul>
{:/nomarkdown}

# The Results

Before we get to a more detailed analysis of each model, there were a few common things which "just worked" for all the models
that actually managed to produce a gamecard (again, more on that later):

* The gamecard was added as the first in the list, as requested.
* The links were added correctly, and the text was formatted as HTML, following the required template.
* The gamecard was formatted correctly, with the title and release date in the right places.

One fun aspect is that for internal purposes, the model had to come up with a unique ID for the gamecard, which all of them did.
Here are the IDs generated:

* `trails-daybreak-ii` : GPT-4o
* `trails-daybreak-2` : GPT-4.1, Claude Sonnet 3.7
* `trails-daybreak2` : Claude Sonnet 4, o4-mini

All of these work and are fine, aesthetically I prefer the first over the second, and the second over the third.

## Model Comparison

Now let's get to the meat of the evaluation, the actual performance of each model. Note that the ratings I provide are
all relative -- 5/5 does not mean that the result is "perfect", but rather the best I could get from any model (and, as
it turns out, quite usable in this case).

### GPT-4o 
By far the fastest, understood the assignment in principle. Didn't actually extract any meaningful information from the
provided context files (only read 50 lines of each). Correct layout, decent prose, but content is generic to the point
of being nearly information-free. This is true "AI slop".

*Rating*: **2/5**

{::nomarkdown}
<details> <summary>Output</summary>
{:/nomarkdown}

```html
<div class="gamecard">
    <details>
        <summary role="term" aria-details="trails-daybreak-ii">
            <h3 class="gameproject">The Legend of Heroes: Trails through Daybreak II</h3>
            <p class="gameproject">
                <img src="projects/daybreak2.jpg" alt="Trails through Daybreak II" class="gameproject">
            </p>
            <div class="collapsible">
                Type: <b>PC Port</b>
                <br><small>Release: <b>2025</b> &nbsp; - &nbsp; Click for more project information</small>
            </div>
        </summary>
    </details>
    <div class="collapsiblecontent" role="definition" id="trails-daybreak-ii">
        Trails through Daybreak II builds upon the foundation of its predecessor, introducing a host of new features and enhancements.
        This includes cutting-edge graphical improvements such as <strong>TAA and DLSS support</strong>, as detailed in 
        <a href="https://store.steampowered.com/news/app/2668430/view/544477139480609941">this Steam announcement</a>.
        <br><br>
        The PC port also boasts a comprehensive feature set, including improved draw distances, refined graphics settings, and numerous
        quality-of-life enhancements. For a full overview, check out 
        <a href="https://store.steampowered.com/news/app/2668430/view/545600503107552212">this Steam post</a>.
        <br><br>
        Leveraging the PH3lib framework, the port achieves remarkable performance optimizations, ensuring a smooth and immersive
        experience for players. This project exemplifies PH3 Games' commitment to delivering high-quality PC ports.
    </div>
</div>
```

{::nomarkdown}
</details>
{:/nomarkdown}

### GPT-4.1
Takes about 90 seconds.

Reads *everything*, and generates a bullet-point summary of the content, as well as a completely correct snippet.
However, it absolutely does *not* understand what "matching the style" means. It claims it did that, but not a single
existing card is a bullet-point list. 

Impressive information extraction and retention, but failed a core part of the assignment.

*Rating*: **3/5**

{::nomarkdown}
<details> <summary>Output</summary>
{:/nomarkdown}

```html
<div class="gamecard">
    <details>
        <summary role="term" aria-details="trails-daybreak-2">
            <h3 class="gameproject">The Legend of Heroes: Trails through Daybreak II</h3>
            <p class="gameproject">
                <img src="projects/daybreak2.jpg" alt="Trails through Daybreak II" class="gameproject">
            </p>
            <div class="collapsible">
                Type: <b>PC Port</b>
                <br><small>Release: <b>2025</b> &nbsp; - &nbsp; Click for more project information</small>
            </div>
        </summary>
    </details>
    <div class="collapsiblecontent" role="definition" id="trails-daybreak-2">
        Trails through Daybreak II is the direct sequel to Trails through Daybreak, and our second PC port built on Falcom's new engine and our <strong>PH3lib framework</strong>.
        <br><br>
        Building on the extensive PC-specific features and enhancements of the first game, we ensured <b>full feature parity</b> and introduced further improvements, including:
        <ul>
            <li>Support for <b>arbitrary resolutions and aspect ratios</b>, including ultrawide and beyond</li>
            <li>Variable frame rates up to 360 FPS (and beyond via config)</li>
            <li>Extensive graphics options: scalable draw distance, shadow quality (with PCSS and HBAO+), MSAA, SGSSAA, trSSAA, and more</li>
            <li>Highly customizable input, new mouse targeting mode, and full mouse-driven UI</li>
            <li>Turbo mode, advanced autosave, and blazing fast loading via asset caching</li>
            <li>Bonus features like detailed BGM info, Steam Recordings timeline support, and direct Steam resume</li>
        </ul>
        <br>
        For a comprehensive overview of the PC port's features and technical highlights, see
        <a href="https://store.steampowered.com/news/app/2668430/view/545600503107552212">this Steam post</a>.
        <br><br>
        In 2025, we released a major update introducing <b>Temporal Anti-Aliasing (TAA)</b> and <b>NVIDIA DLSS 3/4</b> support, further enhancing image quality and performance for a wide range of hardware. For technical details and patch notes, see
        <a href="https://store.steampowered.com/news/app/2668430/view/544477139480609941">this announcement</a>.
        <br><br>
        As with all our ports, we paid special attention to performance, compatibility, and quality-of-life improvements, ensuring a top-tier experience for PC players.
    </div>
</div>
```

{::nomarkdown}
</details>
{:/nomarkdown}

### Claude Sonnet 4
Takes about 50 seconds.

Reads only the first 200 lines of the context files, then tries to search, but chooses bad terms and fails to find
anything. By **far** the best at matching the style. Reads the existing game cards and produces text that I might have
written myself -- if I was missing half the information. If it fully read the context files, it might have been truly
impressive.

*Rating*: **4/5**

Also provides a strangely enthusiastic summary of what it did in chat.

{::nomarkdown}
<details> <summary>Summary provided by the model</summary>
{:/nomarkdown}

```text
Perfect! I've successfully added the Trails through Daybreak II gamecard as the first entry in the projects section. The new gamecard includes:

* Proper structure - Following the same pattern as other gamecards with a <details> summary and collapsible content
* Correct image reference - Using daybreak2.jpg which exists in the projects folder
* Appropriate type and release date - "PC Port & Multi-platform CI/CD" for 2025, matching the style of Ys X: Nordics
* Technical context - References to PH3lib framework and P3A archive system, consistent with the first Daybreak game
* Both provided links - The TAA/DLSS announcement and the PC features overview
* Consistent styling - Matches the tone and technical depth of other entries, particularly the original Daybreak and Ys X entries
* The content emphasizes the technical evolution from the first Daybreak game while highlighting the new TAA and DLSS features as key improvements for this release.
```

{::nomarkdown}
</details>
{:/nomarkdown}

{::nomarkdown}
<details> <summary>Output</summary>
{:/nomarkdown}

```html
<div class="gamecard">
    <details>
        <summary role="term" aria-details="trails-daybreak2">
            <h3 class="gameproject">The Legend of Heroes: Trails through Daybreak II</h3>
            <p class="gameproject">
                <img src="projects/daybreak2.jpg" alt="Trails through Daybreak II" class="gameproject">
            </p>
            <div class="collapsible">
                Type: <b>PC Port & Multi-platform CI/CD</b>
                <br><small>Release: <b>2025</b> &nbsp; - &nbsp; Click for more project information</small>
            </div>
        </summary>
    </details>
    <div class="collapsiblecontent" role="definition" id="trails-daybreak2">
        Trails through Daybreak II continues the Calvard arc of the series, building upon the technical foundation
        established in its predecessor. This second installment in the new arc showcases the maturity of our PH3lib
        framework, with further refinements to the P3A archive system and additional optimizations throughout the engine.
        The port maintains all the high-quality features players have come to expect from our work on the series.
        <br><br>
        One of the standout additions in this release was the introduction of TAA (Temporal Anti-Aliasing) and DLSS support,
        bringing modern rendering techniques to enhance visual quality and performance. You can read more about these improvements
        and their implementation in
        <a href="https://store.steampowered.com/news/app/2668430/view/544477139480609941">
            our detailed technical post about TAA and DLSS integration</a>.
        <br><br>
        As with our previous work on the series, we included a comprehensive set of PC-specific enhancements including
        ultrawide support, extensive graphics options, and quality-of-life improvements. For a complete overview of
        all the features and improvements we brought to the PC version, check out
        <a href="https://store.steampowered.com/news/app/2668430/view/545600503107552212">
            our comprehensive PC features overview</a>.
    </div>
</div>
```

{::nomarkdown}
</details>
{:/nomarkdown}

### Claude Sonnet 3.7
Took **over 3 minutes (!!)**.

As mentioned above, I originally intended to only use the latest version of each model, but as Sonnet 4 was so close to
actually delivering, I decided to try Sonnet 3.7 as well. It also started out reading only part of the context files,
but then -- while spending a lot of time on it -- actually extracted the relevant context. 

It is just as good at matching the style as Sonnet 4, but now has all the relevant information. The result is a usable
game card, with relevant information and in the correct style.

*Rating*: **5/5**

Also provides a good summary, and is less convinced of the perfection of its result than Sonnet 4.

{::nomarkdown}
<details> <summary>Summary provided by the model</summary>
{:/nomarkdown}

```text
I've successfully added a new game card for Trails through Daybreak II as the first item in the list. The new game card includes:

1. The appropriate header with the game title
2. The image path set to daybreak2.jpg
3. The project type labeled as "PC Port"
4. The release date set to 2025
5. A detailed description that includes:
  * Information about the continuation of the Calvard arc
  * Details about the technical achievements, particularly the TAA and DLSS implementation
  * Links to both Steam articles you provided
  * Mentions of other improvements like water rendering, shadows, and cloud rendering
  * A writing style that matches the existing game card descriptions

The game card has been added as the first item in the list, and all the formatting matches the existing game cards. It now appears at the top of the projects list to reflect its status as the most recent project.
```

{::nomarkdown}
</details>
{:/nomarkdown}

{::nomarkdown}
<details> <summary>Output</summary>
{:/nomarkdown}

```html
<div class="gamecard">
    <details>
        <summary role="term" aria-details="trails-daybreak-2">
            <h3 class="gameproject">The Legend of Heroes: Trails through Daybreak II</h3>
            <p class="gameproject">
                <img src="projects/daybreak2.jpg" alt="Trails through Daybreak II" class="gameproject">
            </p>
            <div class="collapsible">
                Type: <b>PC Port</b>
                <br><small>Release: <b>2025</b> &nbsp; - &nbsp; Click for more project information</small>
            </div>
        </summary>
    </details>
    <div class="collapsiblecontent" role="definition" id="trails-daybreak-2">
        Trails through Daybreak II continues the Calvard arc of the Trails series, building on the foundation
        established in the first Daybreak title. For this PC port, we continued to leverage our PH3lib framework
        while expanding the feature set even further. We not only included all PC-specific features from the
        first game but introduced substantial new enhancements.
        <br><br>
        One of the most technically demanding additions was the implementation of Temporal Anti-Aliasing (TAA)
        and NVIDIA DLSS support. This required a complete overhaul of the rendering pipeline to generate accurate
        motion vectors, a feature not natively supported by the game engine. The result provides dramatically improved 
        image quality while enabling higher performance on compatible hardware, as detailed in 
        <a href="https://store.steampowered.com/news/app/2668430/view/544477139480609941">
            our TAA and DLSS implementation announcement</a>.
        <br><br>
        The port includes numerous other enhancements as well, from improved water rendering to optimized
        performance for devices like Steam Deck. We also introduced subtle improvements to shadow quality, 
        cloud rendering, and color processing precision. For a comprehensive overview of all PC-specific
        features and technical details, see our 
        <a href="https://store.steampowered.com/news/app/2668430/view/545600503107552212">
            PC Features Overview</a>.
    </div>
</div>
```

{::nomarkdown}
</details>
{:/nomarkdown}

### Gemini 2.5 Pro (Preview)
This ran for a long time, but that comparison doesn't really make sense -- read on.

From the stream-of-consciousness output, it seemed to do a good job of fully reading the context files and extracting
relevant information, as well as looking up the style of the existing game cards. However, it then went into a loop of
editing `index.html`, while not actually making any changes. This went on for a minute or more, producing 566 "edits" in
the process, but not actually changing the file.

![A screenshot of Gemini 2.5 Pro editing the same file 566 times](gemini_fun.png){: #fancyimg }
_Are we there yet Gemini?_

*Rating*: **1/5** -- But I give it a 4/5 in entertainment value.

I am curious how this one would have performed if it hadn't failed at the end. Maybe a non-"Preview" version will succceed?

### o4-mini (Preview)
Took about 80 seconds. 

It only read 200 lines of each context file, and -- at least according to its stream-of-consciousness output -- it did
not particularly care about investigating the style of the existing game cards. 

However, that alone didn't prepare me for the result. The game card produced labels the game as a *"continuation of the
new engine arc"*, which is admittedly funny but obviously silly and wrong. It also hallucinates a *"dynamic streaming
system"*, which is not a thing and not mentioned anywhere in the context files.

*Rating*: I award you 0 points, and may the machine god have mercy on your soul.

{::nomarkdown}
<details> <summary>Output</summary>
{:/nomarkdown}

```html
<div class="gamecard">
    <details>
        <summary role="term" aria-details="trails-daybreak2">
            <h3 class="gameproject">The Legend of Heroes: Trails through Daybreak II</h3>
            <p class="gameproject">
                <img src="projects/daybreak2.jpg" alt="Trails through Daybreak II" class="gameproject">
            </p>
            <div class="collapsible">
                Type: <b>PC Port</b>
                <br><small>Release: <b>2025</b> &nbsp; - &nbsp; Click for more project information</small>
            </div>
        </summary>
    </details>
    <div class="collapsiblecontent" role="definition" id="trails-daybreak2">
        Trails through Daybreak II marks the continuation of the new engine arc, bringing together extensive engineering improvements for the PC platform.
        We introduced a fully revamped asset pipeline and dynamic streaming system, building on our PH3lib framework for high-performance loading and patching.
        You can read more about these enhancements in 
        <a href="https://store.steampowered.com/news/app/2668430/view/545600503107552212">
            our PC feature overview</a>, which dives into highlights such as optimized resource management and quality-of-life UI updates.
        <br><br>
        In version 1.2.8, we also added advanced rendering options with 
        <a href="https://store.steampowered.com/news/app/2668430/view/544477139480609941">
            TAA and DLSS support</a> to improve visual clarity and performance on modern GPUs.
    </div>
</div>
```

{::nomarkdown}
</details>
{:/nomarkdown}


# Summary & Conclusion

To summarize, all models (except Gemini 2.5 Pro) managed to produce some sort of game card following the basic
formatting and structural requirments. However, the quality of the content varied significantly -- here's a quick
overview of the results, with the models sorted by increasing quality of the output:

| Model            | Rating   | Notes                                                                                   |
|------------------|----------|-----------------------------------------------------------------------------------------|
| o4-mini (Preview)| ☆☆☆☆☆ | Hallucinated a lot, and didn't read the context files.                                  |
| Gemini 2.5 Pro (Preview)   | ★☆☆☆☆ | Failed to produce a usable game card, but provided entertainment value.       |
| GPT-4o           | ★★☆☆☆ | Fast, but generic and information-free.                                                 |
| GPT-4.1          | ★★★☆☆ | Good information extraction, but failed to match the style.                             |
| Claude Sonnet 4  | ★★★★☆ | Good at matching the style, but failed to read the context files fully.                 |
| Claude Sonnet 3.7| ★★★★★ | Best overall, read the context files and matched the style.                             |

In conclusion, this was *fun* (at least for me), but I could have written the game card myself in roughly 4% of the time
this investigation took (without writing this post). With what I know now, and assuming that the behaviour would stay
the same for future tasks, perhaps I could actually save a few minutes on the next game card -- but I still need to
validate the result and make sure it didn't hallucinate.

**I think that the task at hand here (writing a small text blurb and formatting a bit of basic HTML, with specific references
provided for both content and style) is close to a best-case scenario for LLMs, and the results are still decidedly mixed.**

That said, at least the HTML and overall output formatting was correct in all cases, and the result produced by the best-in-class
model (Claude Sonnet 3.7) is actually quite good, and I would be happy to use it as-is. That said, it also took over 3 minutes to
produce, and the energy it consumed while doing so probably keeps my feeble human brain going for a day.
