---
layout: post
title: My Workflow For Using Git and Git Annex For Data Analysis
categories: [programming,R,git]
---

I've struggled for years with using git or any revision control system
for data analysis. Even though I had a fair amount of prior experience
with git before becoming a Computational Biologist, the differences
between producing software and a data analysis always made me stumble
over the git workflows that I'd used in the past. 

# My Setup

My working setup is maybe a little more elaborate than average, but
the problems I'm dealing with are going to be common to the average
computational biologist. 

* For any given project, I have a single directory housing the
  project, with a bunch of R scripts and data. There's a simple
  Makefile that uses knitr to generate report files and images from
  the R and document sources.

* I have a work computer that's a fairly beefy desktop with plenty of
  computing power and disk space

* I have a personal laptop that's fairly underpowered and always short
  on disk space. The laptop can ssh in to the work computer from home.

* I'm also fortunate to have access to file and computing servers that
  I can use and install software on or have backups.

# My Goals

* Be able to store the history of the work in git. The primary focus
  should be on the R scripts and documents used by knitr. 

* Easy backups of the project directory. Cloning the git repository
  gives me this for free.

* Be able to work on the laptop at home after pulling the current
  tree. I need to be able to manage the disk space used by the project
  on the laptop

Here are the major problems and how I've addressed them up until now.

# Data Management

Software development like git doesn't really tend to deal with large
data files in the way a Computational Biology analysis needs to. As a
result, git by itself isn't great handling them and I've found that it
makes a big difference in my ability to get things done. At the same
time, data management is the fundamental requirement for even starting
an analysis, so if I can't manage my data with git it's a show
stopper. Here's how I dealt with specific classes of data that I use.

## Large Project-Specific Files

There are always multiple large files that belong to a given
project. If nothing else, the central data for the project can be very
large. Synchronizing these files across git repositories can take a
long time, and for any given piece of the analysis I won't need the
majority of the data. Simply storing them in the git repository means
that I have to keep lots of things on my laptop hard drive that I
might not need and slows down repository cloning. My time is limited
at home, so this can be a deal breaker for getting any work done when
I'm not at the lab.

Joey Hess's outstanding [git-annex](https://git-annex.branchable.com/)
is the solution I've picked for dealing with this. Large files can be
checked in to git annex, and the files are symlinked in to the working
directory. git annex stores the files in the .git directory, but
manages the files separately from git's internal objects. You keep all
the benefits of git for those files including history and branching,
without actually keeping the content in git. Critically, it allows you
copy only what files you want in to any given repository clone, and
drop those from the repository that you don't want to keep locally. It
tracks how many actual copies of every file exist across all of your
remote repositories, and which repositories have those files if you
want to download them locally. So I can download only the files I need
for my work on my laptop, and keep things moving.

## Large External Flat Data Files

I've long had a collection of external biological metadata that I've
downloaded for local use. These exist as a shared library of data
outside of any particular project, and are usually a pile of flat
files. For working on places like my laptop, I probably don't need all
of these files at once, and wouldn't have the hard drive space for
them all anyway. Again, I turn to git-annex to manage these files,
using one big repository for the data, and just pull the files I need
for any given analysis when I want them.

## SQL Databases

I've tried setting up both PostgresSQL and MySQL databases to access
some of the more complex data sets that I tend to want joins
across. So far, I don't have a solution for dealing with these, so I'm
in the process of moving them in to SQLite files and managing them
with git-annex the same way I do with the flat files above.

Another approach I'm trying is to make sure I've got a Makefile and a
build script to generate the database tables. My build scripts tend to
use either R and its DBI methods to build the tables, or
Python's [CSVKit](https://csvkit.readthedocs.io). That way you can run
those on a local checkout and reproduce the external SQL
database. Ultimately, I think having both the SQL build system and
SQLite files managed with git-annex is going to be what I'll use.

# Workflow

When distributed revision control systems were the newest hottest
thing in Free Software, it seemed like new workflows were popping up
every few weeks. By exposing so much plumbing to the user, git has
always been incredibly flexible and projects end up relying on
convention to stay organized. So we can do pretty much whatever we
want with git, but what exactly we want to do?

## When Do We Make a Commit?

Hadley Wickham has distilled down the basic process of any analysis
pretty well [here](http://r4ds.had.co.nz/explore-intro.html). We
generally evolve some sort of data import that is largely static
through time, with minor patches. This sort of thing is very well
handled by standard git commits. 

The next phase of the process is the data exploration cycle which
involves some transformation of the data, and modeling and
visualization. In my experience, this cycle is usually very messy,
with multiple types of transformation, visualization, and modeling
going on at any time. Questions are frequently poorly specified at
this point, since you're often just trying to get a feel for the
data. Pieces of this cycle readily blur in to one another, making it
hard to know when to make a commit to revision control.

I don't have a good answer for this yet. My own work is broken up in
to knitr blocks, and committing once per knitr block is a
possibility. Usually though I'll have several knitr blocks
contributing to getting one result, be it a model or plot, and that's
the basic unit of work. The standard way to handle these things with
git is to use topic branches and then merge back to the master when
you're done, possibly after rewriting the history on the topic branch
to make things clean. 

My sense is that I'm going to try my best to use this approach but I'm
skeptical that I can be disciplined enough to do it. The goal will be
to branch per piece of the project, and structure commits based on
some tangible result like a transformed data set or plot. Since I work
alone, I'm not too worried about not sticking to this perfectly. This
would be a bigger deal for teams of people who will likely have their
own needs they have to meet.

## How Do We Cache the Results of Long Computations?

This is something I tried to force git to do for me to save time, and
it ended up not being worth it. I love knitr's ability to cache
computation results and I tried checking both these and the build
intermediates like generated standalone plot images in to git. The
problem is that these files are very large and can change a lot.

Basically I've given up on doing this automatically for computations
that might change a lot, because it's not worth it. For long
computations I'm unlikely to do this a lot, I'll manually store those
files as serialized Rdata files and manage them with
git-annex. Another possibility is to just rsync the build
intermediates around and put that in a cron job and Makefile target.

Given these decisions, I've added the following patterns to my
`.git/info/exclude` file to have git ignore build intermediates by
default:

```
*~
*.html
*.asciidoc
*.md
.Rhistory
.RData
cache
figure
projectlib/man/*.Rd
projectlib/data/*.rda
```

The `projectlib` is the name of the internal R package used for the
analysis, and has a different name for each project. Also, I
use [asciidoc](http://www.methods.co.nz/asciidoc/index.html) instead
of Markdown with knitr to build reports, which is why the .asciidoc is
there.

## If We Don't Store Build Intermediates, What Do We Store?

For my setup I need to store the following in git:

* The build system including Makefiles, javascript, and css to build html files
* A project-specific R package that manages tidying the basic data and
  organizing it, allowing lazy loading and centralization of common
  functions
* R scripts
* Rasciidoc or Rmd files
* Support scripts in languages like (z)sh, python, and perl
* SQL query scripts
* SQL table generation scripts
* Shiny apps attached to the project for broader exploratory visualization

The things I need to store in git-annex include:

* The basic data files either in the project R package or a data folder
* Manually edited versions of plots and images
* Data processed by some external software rather than my own R scripts
* Files I've exported for collaborators that are essentially
  write-only and send
* Rdata files that are explicitly saved by my own scripts to cache the
  results of long computations
* Common data files across projects that go in a separate git repository

# Conclusions

This is still an evolving workflow, and I may publish an update if
things change. But as it stands now it should be workable and flexible
enough for the future.

*TLDR*

* Keep a project in a single git repository with a easy flexible build system
* Manage data files using git-annex, with a separate repository for external data sets
* For SQL databases, generate build scripts and keep them in git, then
  possibly store the results in a file format like SQLite and manage
  them with git-annex
* Use topic branches and history rewriting to handle the messy work of
  exploratory data analysis. Structure commits after history rewriting
  to each build some tangible result as in Hadley's figure, like a
  transformed data set or plot
* Don't bother trying to move most knitr build intermediates between
  machines using git. For truly long computations, explicitly store
  the results in Rdata files using `save()` and manage them with
  git-annex
