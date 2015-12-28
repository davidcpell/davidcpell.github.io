---
layout: post
title: Installing Programs from Source I: What is a CLI?
---

In your software engineering career you may, from time to time, find yourself needing to install software that will not become part of your application (like a Ruby library/gem), but will be used in some way to aid in the process of creating your application. Two examples of such software that anyone who has completed a Ruby on Rails tutorial might be familiar with are Git, a version control utility, and Heroku Toolbelt, which lets you manage apps that have been deployed to Heroku from your command prompt. 

These applications are known as "command line utilities," which means that, once installed on your system, they can be used by typing commands from the command prompt. The commands that these applications supply (e.g. `git commit` or `heroku create`) are known collectively as "command line interfaces", or CLIs. This is similar to the concept of an Application Programming Interface, or API. For more on APIs, see my series on [Mapping JSON to ActiveRecord with Representable](http://bit.ly/1MPslCO). 

### Installing from Package Managers / from Source

If you come from a Windows background, you may be familiar with installing new software by downloading a certain file that you can double-click to kick off the installation process. Unix-based operating systems such as Linux and Mac OSX have their own command line utilities, often referred to as "package managers," to streamline the process of installing new software. If you're using some form of Linux, you may have already used the AptGet utility for this, or at least seen it mentioned. Homebrew is a common Mac equivalent. These utilities allow the user to type something as simple as `brew install [program]` or `apt-get install [program]` at their prompt and install new software with little, and in many cases no, further effort: 

```bash
# at my shell prompt

▶ brew install the_silver_searcher
==> Installing dependencies for the_silver_searcher: pcre
==> Installing the_silver_searcher dependency: pcre
==> Downloading https://homebrew.bintray.com/bottles/pcre-8.38.yosemite.bottle.tar.gz
######################################################################## 100.0%
==> Pouring pcre-8.38.yosemite.bottle.tar.gz
🍺  /usr/local/Cellar/pcre/8.38: 146 files, 5.9M
==> Installing the_silver_searcher
==> Downloading https://homebrew.bintray.com/bottles/the_silver_searcher-0.31.0.yosemite.bottle.tar.gz
Already downloaded: /Library/Caches/Homebrew/the_silver_searcher-0.31.0.yosemite.bottle.tar.gz
==> Pouring the_silver_searcher-0.31.0.yosemite.bottle.tar.gz
==> Caveats
Bash completion has been installed to:
  /usr/local/etc/bash_completion.d
  ==> Summary
  🍺  /usr/local/Cellar/the_silver_searcher/0.31.0: 8 files, 116K
```

Wow, that was easy!

If you've ever read a software installation readme, you may have seen the option to install "from source" as an alternative to using a package manager:

<center><img src="images/build_from_source/ag_build_from_source.png" alt="The Silver Searcher - Build from Source (Tarball)"></center>

Basically this means that you would download a copy of the source code (all tied together into one file) and go through the steps of extracting and installing the software by hand rather than have a package manager do it for you. 

### Why Install from Source?

To begin with, it's possible that you may find yourself wanting to install software that isn't available through a package manager. Most of the popular tools that you'll need, especially as you're just getting started in your career as a developer, _will_ be, but it's always possible that you'll need one that isn't. However, a more likely scenario is that you might find yourself working on a system where you don't have permission to add or remove applications that are available system-wide. Notice what happens when I try to use AptGet to install `the_silver_searcher` on a remote machine that is shared among my team at work:

```bash
# on a remote, shared machine

▶ apt-get install the_silver_searcher
E: Could not open lock file /var/lib/dpkg/lock - open (13: Permission denied)
E: Unable to lock the administration directory (/var/lib/dpkg/), are you root?
```

Here AptGet wants me to preface the installation command with `sudo` in order to override the permission error. But since I don't have the root password, this isn't an option. Knowing how to install the program from source will allow me to make sure that all of the files are in my personal directories, thereby avoiding security-related errors. 

In Part II of this series, I'll cover two core concepts for better understanding what exactly we're doing when we install something from source: executable files and the `$PATH` global variable.
