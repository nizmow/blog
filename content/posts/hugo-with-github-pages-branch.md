---
title: "Publishing your Hugo blog on Github Pages with a 'gh-pages' branch"
date: 2020-01-20T19:13:23+01:00
categories:
- devops
---

I started this blog with Hugo because it's super fast and simple, and hosting with Github Pages because it's... super fast and simple. Until now I was building manually and publishing all pages manually to a separate repository, but that was a) tedious and b) limiting my options for eg: writing new posts on my iPad. Today I had some time on my hands and decided to automate this process.

### Using a `gh-pages` branch

First step was to switch away from a separate Github Pages repository to using a `gh-pages` branch of the current repository, since having a separate repository seemed messy. Using a separate branch like this seems a bit counterintuitive when you're thinking in the standard model of a git repository looking like a big tree, since we want this branch to live on its own with no relation to `master`. Git has a name for this type of branch: an _orphan_[^1]. You can set up an empty orphan branch in an existing repository fairly easily:

1) Make sure your `master` branch is clean (everything is committed).
2) Create your `gh-pages` orphan branch: `git checkout --orphan gh-pages`.
3) Clear out this branch: `git reset --hard`. It's an orphan, so everything should be gone, but maybe follow up with `rm -rf *` for stubborn stains.
3) We want to push to origin, but we need a commit. Make an empty one with `git commit --allow-empty -m "Initial gh-pages commit"`.
4) Push! `git push origin gh-pages`.
5) Return to master: `git checkout master`.

You've now got an empty `gh-pages` branch. Now might be a good time to go into your repository settings on Github and configure this branch for hosting.

### Manually publishing to the `gh-pages` branch

With this done, you can now publish a Hugo build of your site from your `publish` folder to your new `gh-pages` branch. For this we use [`git worktree`](https://git-scm.com/docs/git-worktree). Later we can automate this, so it's probably not a good idea to push any changes.

1) Make sure your `master` branch is clean.
2) Remove your `public/` directory and add it to `.gitignore`.
3) Run `hugo` to build and publish your site. You can now add and commit your published changes: `cd public && git add . && git commit -m "gh-pages commit!"`.
4) Push! `git push origin gh-pages`.

If all is well, you should be able to see your published content at the Github Pages site you configured earlier.

If you intend to automate things, though, it's probably a good idea to get rid of the worktree just added (`rm -rf publish/ && git worktree prune`) and get started with Github Actions...

### Automate with Github Actions

Github Actions is a CI platform built into Github, and contributor [crazy-max](https://github.com/crazy-max) has built a whole lot of useful Actions to help with publishing Github Pages. There are a few steps to get Actions going on your repository, a good start is clicking the "Actions" tab and following the instructions. You can see the Actions definitions file I've used to automate publishing of this blog [here](https://github.com/nizmow/blog/blob/master/.github/workflows/publish.yml)... and if you poke through the commit history you can see some messy trial and error while I try to get things running right.

With that done, you can push your Hugo code `master` and in a few seconds your blog will be updated for you. Who needs Wordpress?

[^1]: you could probably do some really wild things with this. Mega-mono-repo?
