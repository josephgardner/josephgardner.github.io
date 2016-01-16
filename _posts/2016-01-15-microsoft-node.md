---
layout: post
title: Micrsoft Node: Put a Fork in It
comments: true
---

I continue to be impressed with Microsoft's new commitment to OSS. 

The latest addition being their JavaScript engine, [ChakraCore](https://blogs.windows.com/msedgedev/2016/01/13/chakracore-now-open/). 

Even more interesting is that they [forked node](https://github.com/Microsoft/nodejs-guidelines) and are working on a [pull request](https://mobile.twitter.com/gauravseth/status/674393822420271104) to have node officially support ChakraCore. They were able to accomplish this quickly by creating a V8 shim that reroutes API calls to their engine. Flip of a switch!

If you are curious just how difficult it can be to open source an existing product, "[So You’ve Decided To Open-Source A Project At Work. What Now?](https://www.smashingmagazine.com/2013/12/open-sourcing-projects-guide-getting-started/)" nicely describes all the things you'll need to consider before taking the plunge and pushing to github. 