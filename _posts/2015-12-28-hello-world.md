---
layout: post
title: Let's start a blog
comments: true
---

Reddit recently [announced](https://www.reddit.com/r/redditdev/comments/3xdf11/introducing_new_api_terms/) they are giving developers until March 17th to move to OAuth2. Originally, the deadline was [August 3rd](https://www.reddit.com/r/redditdev/comments/2ujhkr/important_api_licensing_terms_clarified/) (oops), but I guess they decided to push it back (yay.) My iOS app [Subdit](https://itunes.apple.com/us/app/subdit-r-random-browser-for/id913001545?mt=8) uses [RedditKit](https://redditkit.com/) which at the time of this writing does not support OAuth2 (but has an [open issue](https://github.com/samsymons/RedditKit/issues/61) to add it.)

In addition to supporting OAuth2, I have a couple new features in the works that need a little polish. Having chosen to use storyboards, I feel like developing the UI is prohibitively slow. Also, the primary design consists of dynamically sized table cells which doesn't really work that great on iOS. I mean, it works, but is incredibly cumbersome having to calculate each height based on the rendered text context and any chrome.

I also don't have an Android app.

So I've decided to rewrite Subdit as a hybrid app on the [ionic framework](http://ionicframework.com/). Dynamic cell heights are trivial in html, plus I'm excited about Angular and have used ionic in the past.

My goal is parity with the current iOS app and releasing an Android version.

So what does any of this have to do with writing a blog? Well, apparently I'm using npm on OSX wrong.

### Sudo all the things.

The first step is to update my install of the ionic commandline tools. The README says OSX users will need to **sudo** install it globally. But then you notice the following nagging remark

> or users can [setup proper file permissions on OSX for npm](http://www.johnpapa.net/how-to-use-npm-global-without-sudo-on-osx/) to install without sudo.

It is these types of discoveries along the way that I like to share with colleagues, usually through email. Though, for posterity I'd rather have somewhere to easily bookmark interesting articles and capture ideas.

So let's start a blog.

### What was I doing again?

Oh, right. Ionic. Well, instead of updating the commandline tools, I now have several tabs open for [creating a blog](http://joshualande.com/jekyll-github-pages-poole/), [github pages](https://pages.github.com/), [jekyll](https://jekyllrb.com/), and [setting up a separate environment for ruby on OSX](https://www.iocaine.org/posts/getting-started-with-github-pages-and-jekyll-on-osx.html) because apparently I'm installing gems wrong too.

If you are reading this, then we've successfully **started a blog**!

Oh, and I'll also list movies I've recently watched that I liked.
