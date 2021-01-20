---
layout: post
title:  "How to setup GitHub Pages"
date:   2021-01-19 17:50:00 +0100
categories: github tutorial
---
OK, this is my first post on GitHub Pages.

I was looking for an easy way to create a personal blog about software and programming, and [GitHub Pages](https://pages.github.com/) was a natural choice.

At first, to be honest, I expected GitHub Pages to be an idiot-proof solution. However, it turned out to be sort of a nightmare. The problems are related to a mix of bad documentation and out-of-date software dependencies.

You can find the long story in [The Beginner's Guide to Bundler and Gemfiles](https://www.moncefbelyamani.com/the-beginner-s-guide-to-bundler-and-gemfiles/). In this post I will try to help you setup your GitHub Pages site as fast as possible.

### Prerequisites
In the following tutorial I will assume that:
- you already have a GitHub account, and you have a working setup (git, SSH keys etc...) on your computer
- you know the basics of how to use a terminal (AKA command prompt)
- you have `ruby` and `bundler` commands installed on your computer

### Step-by-step tutorial
First of all, forget [the official documentation](https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll) ;-)

Assuming for example that your GitHub user name is `cooluser`, you have to:
- create a public repository on your GitHub account named `cooluser.github.io` (otherwise it won't work)
- open the terminal, `cd` to some path and run e.g. `git clone git@github.com:cooluser/cooluser.github.io.git`

I have created a *starter kit*, i.e., a repository containing all the files you need to get started. Proceed in the following way:
- download [this ZIP file](https://github.com/guidoman/gh-pages-jekyll-starter/archive/main.zip) and uncompress it somewhere
- copy **all** files from the uncompressed ZIP directory to the *cooluser.github.io* directory created above by `git clone`. All files including hidden one (those starting with a `.`) must be copied. Use for example `cd -R <src dir> somepath/cooluser.github.io`

You should now `cd` to your local directory `cooluser.github.io` and run:
```
bundle update
```
This will take a while. It will install all required Ruby packages into sub-directory *vendor/bundle*.

Once you are done with the previous command, you are ready to try your GitHub Pages site locally. Run:
```
bundle exec jekyll serve
```
and open [http://127.0.0.1:4000](http://127.0.0.1:4000). You should see the default generated site!

Now commit your changes and push them to GitHub. Open [https://cooluser.github.io/](https://cooluser.github.io/). In a few moments your site will be available.

### Where to go next
Now you have to start creating content and customize your site:
- customize the [Minima](https://github.com/jekyll/minima) template or choose a different one
- write new posts by simply creating Markdown files in the *_posts* directory
- publish your changes by simply `git commit` and `git push` again