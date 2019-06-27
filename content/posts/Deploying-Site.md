---
title: "Deploying Hugo site to GitHub Pages with Azure Pipelines"
date: 2019-06-10T17:39:47+10:00
draft: false
toc: false
images:
tags:
  - GitHub Pages
  - Azure Pipelines
  - Azure DevOps Services
  - Docker
  - Hugo
---

## Summary

The intent is to deploy a [Hugo] site hosted on [GitHub] to [GitHub Pages]. We will be using [Azure Pipelines] as the CI tool to build the site using [Docker] and then push the compiled site back to GitHub for deployment.

## Prerequisites

- Have a functioning Hugo Site
  - <https://gohugo.io/getting-started/quick-start/>
  - Ensure that any themes are added via submodule
- Have a GitHub account
  - <https://github.com/join>
- Basic git experience
  - <https://git-scm.com/book/en/v1/Getting-Started>
- Basic Docker experience
  - <https://docs.docker.com/get-started/>

## Process

### Setup Repositories

The first thing we want to do is to create two GitHub repos. One will contain the source code for the website and the other will contain the compiled and minified code to drive the GitHub Pages site.

For the source code I went with a repo name of `rob-mccloud` since the site will be available at <https://rob-mc.cloud>. The name of this repo is entirely up to you. Check in your site code except any files in `/resources/_gen` or `/public`. Be sure not to add `/public` to the `.gitignore` we will need this folder later.

The GitHub Pages repo on the other hand has a strict requirement in naming. As stated in the [doco][GHPagesDoco], the repo name must be in the form `<username>.github.io` The casing does not matter. My repo is called `robfaie.github.io` despite my username being `RobFaie`. I chose to make this repo private, but it doesn't matter much.

### Adding the GitHub Pages repo as a submodule

When Hugo builds the result is placed in the `public` folder. We will be adding our GitHub Pages repo as a submodule in that folder so that we can simply build and then push the submodule change to deploy. To do this we use the following command from the top of your source code repo.

```bash
git submodule add -b master ../<username>.github.io.git public
```

Note that we are using a relative path for the repo. This is important so that Azure Pipelines uses the same creds for both repos. Make sure to check-in the submodule update.

### Building Locally

Let's build the site locally to check things out and make sure we haven't made any mistakes up to this point. Up until now you have likely been calling `hugo server -D` to serve your site locally. The `-D` bit will build the draft pages which you don't build for the production deployment, so if your pages are still in draft, publish them by setting `draft: false` in the metadata. Now run the following command:

```bash
hugo --gc --minify
```

This will build the site, minify the results, and run some garbage collection. Check the output in the terminal to check that you have built the number of pages you were expecting and have a flick through the `public` folder to make things look right.

Next let's do it again but using docker which we will be using in the Azure Pipeline. I will be using the [jguyomard/hugo-builder] image from dockerhub. It's a simple image that uses alpine linux as a base which is nice and lightweight. Delete the public folder and run the following:

```bash
docker run --rm -v $PWD:/src jguyomard/hugo-builder hugo --gc --minify
```

### Deploy locally

Now that we have our site built we can do a test deployment to iron out any issues with the submodule. Navigate into the `public` folder and give it a push:

```bash
git add .
git commit -m "Local Deployment Test"
git push origin master
```

After a minute your site should be live at `<username>.github.io` If you have set a custom domain set in your hugo config, your site will likely have broken css and assets. We'll fix that up when we set up the custom domain.

### Creating an Azure DevOps Organization

We're going to head over to <https://aex.dev.azure.com/signup/> to setup an Azure DevOps Services Organization. We can click "Start free with GitHub" to create one with our existing GitHub account. Once the org is setup, create a project for the website using the "Create Project" in the upper right hand corner.

### Create Azure Pipeline

From our ADOS project, select 'Pipelines' => 'Builds' from the left side bar and then 'New build pipeline' from the 'New' drop down menu. We're going to be going with the 'github (YAML)'.

Select your source code repo from the list. During the OAuth propts that follow select both the source code repo and the GitHub Pages repo when giving access to repos.

On the 'Configure' step, choose 'Starter pipeline'. This will create an `azure-pipelines.yml` file and check it into your repo for you. Alternatively you can create the file yourself and at this step choose 'Existing Azure Pipelines YAML file'.

### The Pipeline Yaml file

The first thing we want in the pipeline configuration file is the trigger. The whole point of going through this effort is so that we don't need to anything other than push code to GitHub for the live website to get updated.

```Yaml
trigger:
- master
```

We're going to be using Ubuntu 16.04 for the build agent since it comes with Docker and there we're not doing anything more than a docker command and a few git commands.

```Yaml
pool:
  vmImage: 'ubuntu-16.04'
```

The first step we're going to configure the checkout step. This step checks out the source code from git and will run regardless of whether it is specified in the pipeline configuration file, but by putting it in here we can change the behaviour slightly.

In this case we want to checkout submodules so we get our Hugo theme. Doing a clean checkout takes a little longer but ensures that we don't have any assets siting around from last build. Lastly we request that the credentials are persisted. This is important for when we want to push to our submodule repo.

```Yaml
steps:
- checkout: self
  clean: true
  submodules: recursive
  persistCredentials: true
```

Now comes the meat of the pipeline. Firstly we want to grab the credentials that Azure Pipelines has left for us and store it in a variable for later. Inside the public folder we checkout master so that we aren't on a detached head and have the latest commit for clean diffs. I have then chosen to delete all contents so that any removed posts or tags are cleaned up fully.

Next we issue the same Docker command as earlier simply replacing the `$PWD` in the volume argument with `$(System.DefaultWorkingDirectory)` which points to the root of our source code repo.

All that is left is to commit the changes and push them. We need to provide a `user.name` and `user.email` for git to use in the commit. I've chosen to use the built in pipeline variables `$(Build.RequestedFor)` and `$(Build.RequestedForEmail)`. In order for Azure Pipelines to be able to do the push it needs to be given the credentials we stored earlier to authenticate the push.



```Yaml
- script: |
    AUTH=$(git config http.$(Build.Repository.Uri).extraheader)
    cd public
    git checkout master
    rm -rf *
    docker run --rm -v $(System.DefaultWorkingDirectory):/src jguyomard/hugo-builder hugo --gc --minify
    git add .
    git reset -- CNAME
    git reset -- README.md
    git -c "user.name=$(Build.RequestedFor)" -c "user.email=$(Build.RequestedForEmail)" commit -m "CICD: $(Build.BuildNumber)"
    git -c http.extraheader="$AUTH" push
```

Putting all of that together we get the following full pipelines configuration file:

```Yaml
trigger:
- master

pool:
  vmImage: 'ubuntu-16.04'

steps:
- checkout: self
  clean: true
  submodules: recursive
  persistCredentials: true

- script: |
    AUTH=$(git config http.$(Build.Repository.Uri).extraheader)
    cd public
    git checkout master
    rm -rf *
    docker run --rm -v $(System.DefaultWorkingDirectory):/src jguyomard/hugo-builder hugo --gc --minify
    git add .
    git reset -- CNAME
    git reset -- README.md
    git -c "user.name=$(Build.RequestedFor)" -c "user.email=$(Build.RequestedForEmail)" commit -m "CICD: $(Build.BuildNumber)"
    git -c http.extraheader="$AUTH" push
```

One of the nice things about setting up the Azure Pipeline this way is that this file is 100% agnostic of the actual repos and doesn't require any Personal Access Tokens or SSH keys floating around. That makes it dead simple to reuse this on another Hugo site deployed to a GitHub Pages Project Page in the future.

### GitHub Pages Custom Domain.

Setting up the custom domain for Github pages is quite simple. The doco can be found at <https://help.github.com/en/articles/using-a-custom-domain-with-github-pages>

Setup a `CNAME` DNS record in your DNS provider pointing directly at `<username>.github.io`. If you are planning on using an apex domain, you may need to use an `ALIAS`, `ANAME`, or `A` record since apex `CNAME` records are not part of the DNS spec. An apex domain is one without a subdomain, `rob-mc.cloud` vs `www.rob-mc.cloud`. My personal provider [CloudFlare] does `CNAME` flattening, so I was able to add the `CNAME` record and CloudFlare took care of converting it to actual to spec records.

Once the DNS is setup and has had time to propagate (up to 48 hours) the custom domain can be added in GitHub. From the settings page for your GitHub Pages repo, scroll nearly to the end to just above the Danger Zone. Here you can enter your custom domain and hit save. GitHub will redirect `<username>.github.io` to your custom domain and request a [Let's Encrypt] SSL certificate for you. The certificate should only take an hour to be issued.

[jguyomard/hugo-builder]: https://hub.docker.com/r/jguyomard/hugo-builder
[Hugo]: https://gohugo.io
[GitHub]: https://github.com
[GitHub Pages]: https://pages.github.com/
[GHPagesDoco]: https://help.github.com/en/articles/user-organization-and-project-pages
[Azure Pipelines]: https://azure.microsoft.com/en-us/services/devops/pipelines/
[Docker]: https://www.docker.com/
[CloudFlare]: https://www.cloudflare.com/
[Let's Encrypt]: https://letsencrypt.org/
