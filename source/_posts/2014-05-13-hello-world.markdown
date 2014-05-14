---
layout: post
title: "Hello World!"
date: 2014-05-13 00:56:09 -0400
comments: true
categories: blog 
keywords: Octopress, Hello World, subdomain, GitHub, CNAME
description: Introductory post to my blog.
---
This is my [inaugural, requisite](https://en.wikipedia.org/wiki/Hello_world_program) post on my personal blog.

I have chosen to run my blog with [Octopress](http://octopress.org/) rather than WordPress or Blogger. Why? [This blogger](http://decodize.com/html/moving-from-wordpress-to-octopress/) and [this blogger](http://blog.zerilliworks.net/blog/2013/03/16/why-octopress/) do a good job of explaining. Not only does Octopress allow me to source control the entire blog, but also lets me very easily host the site using [GitHub Pages](https://pages.github.com/). 

The guide [here](http://octopress.org/docs/setup/) does a great job of explaining setting up Octopress on GitHub. While I am using GitHub to host the static content I wanted to use my own custom domain. Following the instructions [here](http://octopress.org/docs/deploying/github/#custom_domains) to create a CNAME file and then [adding a simple "CNAME Record" (an alias) on GoDaddy](http://support.godaddy.com/help/article/680/managing-dns-for-your-domain-names#cnames) was all it took for [blog.alexrothberg.com](http://blog.alexrothberg.com) to direct to [cancan101.github.io](http://cancan101.github.io).

###<a name="blog-path"></a>Simplifying the URL
The default behavior for Octopress is to put the blog content in the `/blog/` subdirectory in the URL. Since I am using a subdomain for my namespacing, this proved to be awkward. The URLs started with: `http://blog.alexrothberg.com/blog/`. [This post](http://xit0.org/2013/04/remove-redundant-slash-blog-prefix-from-octopress-website/) offers an easy solution. I also found this [GitHub Issue](https://github.com/imathis/octopress/issues/464) discussing the issue. You can see changes I made [here](https://github.com/cancan101/cancan101.github.io/commit/fa2778c349f7d60a16ed073c17702404d159206b). When done with the edits I also had to [run `rake update_source`](http://octopress.org/docs/updating/).


###Helpful Links
* http://blog.zerosharp.com/clone-your-octopress-to-blog-from-two-places/
* http://learnaholic.me/2012/10/10/deploying-octopress-to-github-pages-and-setting-custom-subdomain-cname/ 
* http://stackoverflow.com/questions/12328828/directory-structure-of-octopress
