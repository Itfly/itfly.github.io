---
title: How to split a big branch into several and send CR requests respectively
layout: post
---

Let’s say we have a big feature branch myFeature that contains a lot of
changes (because we want to test the feature locally end-to-end first
before sending out any CRs). For some reason we need to split these changes into two or more CR requests. For example, a common practice is to send interface changes first. Here's one kind of measure which falls in with this demand.

1. First, we need to create another branch that only contains a subset of
changes:
`git checkout -b myFeatureInterfaces origin/master`

1. If we have granular commits on the myFeature branch , we can
cherry-pick specific commits to our target branch:
`git cherry-pick commitId`

1. If our commits are not very granular and we only want to include
specific files in the first CR, we can checkout them: 
`git checkout myFeature path\myInterface.java`

1. Sometimes we only want to include changes to some parts of the file.
For example, `.config` files may have a lot of changes, but we want to
only include the addition of one specific file. In this case we can pass
the `-p` parameter to the checkout command, then Git will interactively
ask us what changes we want to include in the target branch: `git checkout -p myFeature someConfiguration.config`

While this branch is being code reviewed, we can keep working on CRs for
other parts of the feature. For example we can create a new branch that
will have changes for the implementation CR: `git checkout -b myFeatureImplementation myFeatureInterfaces`


Repeat steps 2 ~ 4. Note that we base the new branch off myFeatureInterfaces and not
origin/master, because the interfaces have not been checked in to master
yet (they are still under code review).
