---
title: "Why you should (or shouldn't) consider MassTransit for distributed applications"
date: 2020-01-17T16:00
draft: true
categories:
- devops
---

I started this blog with Hugo because it's super fast and simple, and hosting with Github Pages because it's... super fast and simple. Until now I was building manually and publishing all pages manually to a separate repository, but that was a) tedious and b) limiting my options for eg: writing new posts on my iPad. Today I had some time on my hands and decided to automate this process.

### Using a `gh-pages` branch

First step was to switch away from a separate Github Pages repository to using a `gh-pages` branch of the current repository, since having a separate repository seemed messy. Using a separate branch like this seems a bit counterintuitive when you're thinking in the standard model of a git repository looking like a big tree, since this branch should just live on its own with no relation to `master`. Git has a name for this type of branch: an _orphan_[^1]. You can set up an empty orphan branch in an existing repository fairly easily:

1) Make sure your `master` branch is clean (everything is committed).
2) Create your `gh-pages` orphan branch: `git checkout --orphan gh-pages`.
3) Clear out this branch: `git reset --hard`. It's an orphan, so everything should be gone, but maybe follow up with `rm -rf *` for stubborn stains.
3) We want to push to origin, but we need a commit. Make an empty one with `git commit --allow-empty -m "Initial gh-pages commit"`.
4) Push! `git push origin gh-pages`.
5) Return to master: `git checkout master`.

You've now got an empty `gh-pages` branch. Now might be a good time to go into your repository settings on Github and configure this branch for hosting. To test things out, you can do 

[^1]: you could probably do some really wild things with this. Mega-mono-repo?

# END

notes:

creating a github pages branch:

* make sure there are no outstanding commits in the repo
* create an orphan branch: `git checkout --orphan gh-pages`
* clear it out: `git reset --hard` (it's an orphan, this will delete everything)
* make an empty commit so you can push: `git commit --allow-empty -m "Initial gh-pages commit"`
* push the empty branch to your repo: `git push origin gh-pages`
* you've got a github pages branch!

updating:

* in MASTER branch: `git checkout master`
* make sure your public/ directory is ignored
* commit any changes
* use worktree to track the gh-pages branch in the public/ subdirectory: `git worktree add -B gh-pages public origin/gh-pages`
* run `hugo` to publish and then add and push your changes to the gh-pages branch: `cd public && git add . && git commit -m "gh-pages commit"`
* if all is well, push! `git push origin gh-pages`
