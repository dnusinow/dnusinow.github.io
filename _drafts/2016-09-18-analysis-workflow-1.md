---
layout: post
title: My Workflow For Data Analysis
categories: [programming,R]
---

Everyone who starts out using a programming language like R for data
analysis begins with the giant script approach. You just keep writing
one big script for an analysis and have it dump graphics or files out
somewhere. If you keep doing analyses though, this becomes unwiedly
quickly. A longer-term project might have lots of smaller tasks that
need to be done, and these tasks usually share some common
features. 

At this point, you've found yourself in the same position as most
software developers, where you have think about how to factor your
work in to components and make it work together consistently. The
problem is that the vast majority of data analysts have no training in
software development and don't know how to manage this. Worse, a lot
of the tools and best practices for software development aren't easy
to adapt to exploratory data analysis because they present very rigid
structures for more mature and understood codebases and problem
domains. So what I found that I needed was a flexible workflow for
data analysis that would be lightweight and very easy to hack on, but
also provide very common use cases that I could trivially automate and
build tools around. My workflow is still evolving (I'll be writing up
improved tooling here as I change things) but what I have has been
working very well for me for the past few years and I don't see myself
switching from the basic model any time soon.

# Some Other Workflows

## Project Template

The first thing I tried
was [Project Template](http://projecttemplate.net/index.html). This
was back a few years ago, but what I liked a lot about project
template were two major featues:

1. Having a clearly specified consistent directory layout. No matter
   what the project was, the overall organization would be consistent.
2. Separation of pre-processing of data from analysis. This ensures
   that data munging, cleaning, and normalization happen in one
   consistent spot and the results get shared between analysis
   scripts.
   
What I disliked about Project Template was that it tried to automate
too many things. It would automatically load data, and I simply didn't
have the memory on my computer to be able to do that for all my data
for all scripts. I also didn't like how many things it tried to
automate for you generally, while also feeling somewhat locked out
from the standard command line UNIX tools that I already knew. So I
gave up on Project Template pretty quickly but I kept the basic parts
and idea of the directory organization. This project has kept on and
appears to have gotten some new features, and it's definitely worth a
look for your own work.

## Projects as R Packages

Hadley Wickham had been expousing this idea on Twitter around the time
I was giving up on Project Template, and Robert M. Flight eventually
did an excellent writeup across two blog posts
([here](https://rmflight.github.io/posts/2014/07/analyses_as_packages.html) and
[here](https://rmflight.github.io/posts/2014/07/vignetteAnalysis.html)). The
basic idea here is that an R package already can build data and
documentation via the vignette system, so why not just use the R
package as the framework for your analysis? It sounded like a great
idea to me at the time. You get all the opinionated directory
structure from Project Template, and building the data clearly
separates the pre-processing from the data analysis that occurs in the
vignettes. Plus you can just install your analysis as a package to
distribute it! Very cool idea.

In practice, what I found was that this was a very restrictive
approach. Any time I had to change the pre-processed data, say as I
was working out how to normalize it (which I have been spending months
on for my current projet, for example) I'd have to do a ton of work
rebuilding vignettes with inflexible tools. If I broke a different
vignette that I didn't care about I had to spend a ton of time fixing
it or moving it out of the build tree. If I was building a package
where every piece had to work, this was great. But when I had to get a
figure for a grant due the next day this was just a huge headache.

Besides this, I had no clear way of specifying work to be done prior
to running a script. If I had to download some data or run some
external script prior to running a script, I wanted that to be in the
build system, and there wasn't an obvious way to do that.

# Inverting the System: One Library For Every Project

After failing with the R Packages as Analysis method, I took a step
back and thought about what packages are actually good for. They're
good for shared data and functions. Any large project is going to have
both of these things. 

For shared data, you'll have your munged, cleaned, and normalized
data, as well as secondary metadata. A huge benefit to R Packages is
that they give you free and easy lazy data loading. You can drop a
bunch of different pieces of data in a package and then only what you
need gets loaded in to memory when you use it. This is a huge benefit
over what Project Template was giving me once upon a time. Sharing the
data between analysis scripts only requires you to load the library.

Beyond that, I use these features for more subtle things that we don't
talk about. One big one is having a shared color palette for an entire
project. I build hundreds to thousands of individual plots for a given
project, and this lets me have consistent color palettes across all of
them, making it easier to interpret. I also have a custom ggplot theme
that lets all my plots look exactly how I want them across a whole
project. This is a surprisingly large benefit in practice.

This should be obvious, but I get shared functions as well. No more
sourcing R files blindly or copy and pasted code between analysis
scripts, if I need to share a function, I move it from an analysis
script in to the library, write a quick description using Roxygen,
rebuild the library using devtools and I'm good to go. It takes about
5 minutes more than copy and paste, but it's fairly trivial. I might
try to automate the process away at some point.

# Tying it Together: Knitr and Make
