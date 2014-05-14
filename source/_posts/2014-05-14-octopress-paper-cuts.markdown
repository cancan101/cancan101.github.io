---
layout: post
title: "Octopress Paper Cuts"
date: 2014-05-14 09:16:22 -0400
comments: true
categories: blog
publish: true
keywords: Octopress, CSS, anchors, post_url, gist, GitHub, alignment, bug, paper cuts,
description: I explain bugs in Octopress and their solutions.
---

Now that I have started writing a few blog posts, I have come across a few ["paper cuts"](https://en.wikipedia.org/wiki/Paper_cut_bug) with the (current) Octopress framework. Some quick background on Octopress: I am currently running the v2.0 version of Octopress which appears to have been released in [July 2011](http://octopress.org/2011/07/23/octopress-20-surfaces/). According to [this blog](http://sasheldon.com/blog/2013/07/07/waiting-for-octopress-2-successor/), the v2.1 version was scrapped in favor of a more significant v3.0. From the [Twitter feed](https://twitter.com/octopress) and the [GitHub page](https://github.com/octopress/octopress), v3.0 seems to be in active development but not yet released. That leaves me with using v2.0 for the time being.

####Social Button Alignment
The first issue that I came across was the vertical alignment of the "social buttons" on the bottom of each post. You can see the Facebook "Like" and "Share" buttons are lower than the Twitter and Google+ buttons:

{% img /images/post_images/2014-05-14-octopress-paper-cuts/facebook-alignment.png Facebook Alignment Issue%}

which was fixed by following [this comment](https://github.com/imathis/octopress/issues/176#issuecomment-6531879).

####Embedded Gist Rendering
The next issue was rendering of embedded [Gists](https://gist.github.com/). The rendered Gist look pretty crappy:

{% img /images/post_images/2014-05-14-octopress-paper-cuts/gist-embed-issues.png Embedded Gist %}

There are a number of people talking about the issue: [here](http://devspade.com/blog/2013/08/06/fixing-gist-embeds-in-octopress/) and [here](https://github.com/imathis/octopress/issues/847). [The solution I ended up using was a combination of these plus some of my own CSS.](https://github.com/cancan101/cancan101.github.io/commit/d30d95694e5e4915b80e0082fb3ef1caac1f021b)

####Meta Keywords and Description on Site Pages
Currently meta tags for keywords and description are only generated on post pages not on site level pages. A solution is offered [here](http://qiang.hu/2013/04/octopress-seo-site-keywords-and-description.html) and a [Pull Request](https://github.com/imathis/octopress/pull/1558).

####Blog in URL Path with Subdomain
I addressed this issue in a [previous post]({% post_url 2014-05-13-hello-world blog-path %}). 

####post_url plugin
In order to use `post_url` to allow intelligent linking between posts, I had to follow [this post](http://www.drurly.com/blog/2012/06/01/octopress-linking-to-other-posts).

Then I ran into this error message:
```
Liquid Exception: Tag '{% raw  %}{%%20post_url%202014-05-13-hello-world%20%}' was not properly terminated with regexp: /\%}/{% endraw %} in atom.xml
```
Following [this comment](https://github.com/davidfstr/rdiscount/issues/75#issuecomment-22607869) fixed that issue.

Since I wanted the ability to use named anchors, I made the following change to the `post_url.rb` file:

```diff
--- plugins/post_url.rb.orig	2014-05-14 13:33:36.938495170 -0400
+++ plugins/post_url.rb	2014-05-14 13:30:26.162496398 -0400
@@ -25,8 +25,10 @@
   class PostUrl < Liquid::Tag
     def initialize(tag_name, post, tokens)
       super
+      post, anchor = post.strip.split
       @orig_post = post.strip
       @post = PostComparer.new(@orig_post)
+      @anchor = anchor
     end
 
     def render(context)
@@ -34,7 +36,11 @@
       
       site.posts.each do |p|
         if @post == p
-          return p.url
+          if @anchor.nil?
+             return p.url
+          else
+             return p.url + "#" + @anchor
+          end
         end
       end
```

I doubt these will be the last of the paper cuts that I uncover, so stay tuned for more fixes. I am starting to think [this applies](http://www.art.net/~hopkins/Don/unix-haters/whinux/your-time.html) to Octopress, but I still enjoy the challenge and the learning experience.
