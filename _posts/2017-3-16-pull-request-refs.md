---
layout: post
title: Reducing work using pull request refs
comments: true
---
>  Refspecs are cool and you should not fear them. They are simple mappings from remote branches to local references, in other words a straight forward way to tell git “this remote branch (or this set of remote branches), should be mapped to these names locally, in this name space.”

[http://blogs.atlassian.com/2014/08/how-to-fetch-pull-requests/](http://blogs.atlassian.com/2014/08/how-to-fetch-pull-requests/)

To checkout a pull request locally:
```sh
git fetch origin +refs/pull-requests/your-pr-number/from:local-branch-name
```

Or add the refspec that will map remote pull requests heads to a local pr name space. You can do it with a config command
```sh
git config --add remote.origin.fetch '+refs/pull-requests/*/from:refs/remotes/origin/pr/*'
git fetch origin
git checkout pr/1
```
