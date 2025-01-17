---
title: "Server-side Git setup for deploying a Laravel web application"
prevFilename: "db"
nextFilename: "git-dev"
date: 2023-07-18
---

# Server-side Git setup for deploying a Laravel web app

{{< deploy-laravel/header >}}
<div class="mt-4 mb-10">
{{< tutorials/navbar baseurl="/tutorials/deploy-laravel" index="about" >}}
</div>

This article shows how to set up a Git repository on your app's server, to which you will push code from your development machine, and how to create a post-receive Git hook to automate (re)deploying your app on every Git push to the server.

## Preview

For orientation, here's the Git and deployment workflow we'll use in this guide:

1. You develop your app on your dev machine.
1. You push code from your dev machine to a dedicated Git repo on your server.
1. A post-receive Git hook automatically copy your app's code to the `/srv/www/` directory from which Nginx will serve your app to the public Web.

{{< details summary="Credit where credit is due" >}}
The Git workflow used in this guide is originally inspired by [Farhan Hasin Chowdhury's guide to deploying a Laravel web app on a VPS](https://adevait.com/laravel/deploying-laravel-applications-virtual-private-servers) (which in turn seems to be based on [J. Alexander Curtis's guide to deploying a Laravel 5.3 app on a LEMP stack](https://devmarketer.io/learn/deploy-laravel-5-app-lemp-stack-ubuntu-nginx/)).
I encourage you to read both guides.
{{< /details >}}

{{< details summary="What if I want to host my Git repo on GitHub?" >}}
No problem, you can do that too, in parallel to the server-side Git setup in this article.
Just set up multiple remotes on your dev machine.
E.g. `origin` (GitHub) and `prod` (server).

As an aside, you could even [base your deployment workflow on GitHub actions](https://stefanzweifel.dev/posts/2021/05/24/deployer-on-github-actions), but that is a separate topic and beyond the scope of this tutorial.
{{< /details >}}

## Install Git

First check Git is installed on your server (it probably will be).
Install if needed:

```bash
# Check if Git is installed (which it probably will be)
laravel@server$ apt list git

# Install Git (you probably won't need to)
laravel@server$ sudo apt install git-all
```

## Create directories

We'll first create a bare Git repository to hold the web app's code on the server.
I'm placing the Git repo in a dedicated `repo` subdirectory of the `laravel` user's home directory, but the location is arbitrary---anywhere on the server would work.

```bash
# Create a folder to hold the Laravel project's Git repo
laravel@server$ mkdir -p ~/repo/laravel-project.git

# Initialize a bare Git repo for project
laravel@server$ cd ~/repo/laravel-project.git && git init --bare
```

Note that we created a *bare* Git repo.

{{< details summary="What is a bare Git repo?" >}}
For our purposes, a bare Git repo is a repo without a working tree.
In fact, a bare repo contains the same files and directory structure you would normally find *inside* the `.git` folder of a standard Git repo.

Why use a bare repo?
Our motivation is simple: since we won't be doing any development work on the server, there is no reason to keep a working tree on the server (you should be editing your app's source code only on your development machine).
There is one more minor convenience: we'll be working with Git hook files as part of our deployment workflow; while a normal Git repo hides these hook files away in the hidden `.git` folder, a bare Git repo keeps them in plain sight, and thus easier to access.

Using bare repos is common practice on servers to which you will push code from a development machine (particularly in collaborative workflows involving many people), but not develop any code on the server itself.
See [this section of the Git Book](https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server) and [this StackOverflow question](https://stackoverflow.com/questions/5540883/whats-the-practical-difference-between-a-bare-and-non-bare-repository) for more on using bare repos;
a deeper appreciation of bare repos requires some familiarity with Git concepts like the working tree, `git checkout`, and remote-tracking branches.

(There would also be nothing terribly wrong with using a standard Git repo on the server---you'd just have an unnecessary working tree taking up space and getting in your way, and Git hook scripts hidden away in the `.git` folder.)
{{< /details >}}

Then create a separate directory in `/srv/www/` from which to serve the app:

```bash
# Create directory to hold app files on server and give ownership to the
# laravel user.
laravel@server$ sudo mkdir -p /srv/www/laravel-project
laravel@server$ sudo chown laravel:laravel /srv/www/laravel-project
```

{{< details summary="Why use `/srv/www`?" >}}
I'm using `/srv/www/` because the [Linux filesystem hierarchy standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html#srvDataForServicesProvidedBySystem) (a standard for how to organize your filesystem on a Linux machine) recommends `/srv` for storing data served by your computer, and the `/srv/www/` subdirectory for web sites and applications served over the World Wide Web.

Note that `/var/www/` is also a common location from which to serve web apps and web sites, and indeed you'll find `/var/www/` used in many tutorials online.
Use whichever you prefer.
{{< /details >}}

Note that both the server-side Git repo and the `/srv/www/` directory are currently empty.
We'll populate them in the next article.

Our workflow will be to push code to the bare Git repo `~/repo/laravel-project.git`, then use a post-receive hook to `git checkout` the `HEAD` of your app's Git repo into the `/srv/www/laravel-project` directory from which you app is served.

{{< details summary="Please translate the last sentence to non-jargon" >}}
For our purposes, the phrase

> `git checkout` the `HEAD` of your app's Git repo into the `/srv/www/laravel-project` directory 

translates to updating the files in `/srv/www/laravel-project/` (from where your app is served) to match the version just pushed to the Git repo.
Or even simpler: copying your app's code from the Git repo into the server directory.

Caveat: this assumes you haven't manually changed the `HEAD` of the Git repo on the server to point to something other than your main branch, in which case you probably know what you're doing anyway.
{{< /details >}}

And why use separate Git and server directories in the first place?
This setup decouples the Git repo (which has your app's entire commit history) from the most recent version of your app being served to the public Web.
Aside from being cleaner in principle than serving your app directly from a Git repo, this considerably simplifies management of Laravel's `.env` file, the PHP `vendor` directory, and the Node.js `node_modules` directory for your production app (none of these files should be placed in a Git repo in the first place).

## Create a post-receive hook

We then just need to create a Git hook to checkout your app from the Git repo to the server directory.
We'll do this with a `post-receive` hook (the exact name is important here), which Git will automatically run after every push to the server.

{{< details summary="What is a Git hook?" >}}
Git hooks are just shell scrips that automatically run in response to specific Git-related events (pushes, pulls, commits, etc.).
They're super useful for automating Git-related tasks.

This guide uses a `post-receive` hook to automatically redeploy your web app on every Git push to the server.
There are many other applications of Git hooks---see [the Git Book](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) for details.
{{< /details >}}

Here's what to do:

```bash
# Change into your Git repo's hooks directory.
# (There will be many sample scripts you can look through for inspiration.)
laravel@server$ cd ~/repo/laravel-project.git/hooks

# Create the post-receive hook script.
# (This exact name is needed for the hook to run after Git pushes.)
laravel@server$ touch post-receive

# Make the post-receive hook executable
laravel@server$ chmod +x post-receive
```


Then open the `post-receive` script for editing, and inside place:

```bash
#!/bin/sh

# Server and Git repo directories
SRV="/srv/www/laravel-project"
REPO="/home/laravel/repo/laravel-project.git"

# Copy app from Git repo to server directory
git --work-tree=${SRV} --git-dir=${REPO} checkout --force
```

{{< details summary="Please translate the command to non-jargon" >}}
In practice, the command in the `checkout` script copies the most recently-pushed version of your app from the Git repo to the server directory.
(See also the earlier note on `git checkout` and `HEAD`.)

We're basically running `git checkout` with a few extra options:

- `--work-tree` specifies where the files should be copied to (into the `/srv/www/` directory from which your app is served)
- `--git-dir` specifies from where the files should be copied (your app's server-side Git repo)
- `--force` ensures the checkout command runs even if you've made local changes in the Git repo (and that it actually copies files instead of just printing tracking information to standard output---try removing the `--force` flag and see what happens).

The work tree and Git directory are foundational Git concepts, but are all too often overlooked by new users. It would be well worth your time to become familiar with them---I suggest [Matt Neuburg's guide](https://www.biteinteractive.com/picturing-git-conceptions-and-misconceptions/) as a human-friendly starting point.
{{< /details >}}

The end result is to automatically copy the most recent version of your app to the server directory `/srv/www/` after every Git push.
We'll give this a try at the end of the next article.

**Next:** The next article covers Git setup on your development machine.

<div class="mt-8">
{{< tutorials/navbar baseurl="/tutorials/deploy-laravel" index="about" >}}
</div>

<div class="mt-8">
{{< tutorials/thank-you >}}
<div>

<div class="mt-6">
{{< tutorials/license >}}
<div>

