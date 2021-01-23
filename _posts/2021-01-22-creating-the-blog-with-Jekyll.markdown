---
layout: post
title:  "Creating the blog with Jekyll and GitHub Pages"
date:   2021-01-22 13:30:34 +0100
categories: jekyll creation how to
---

![banner](/assets/jekyllXgithub_banner.png)

Sooo, let's talk about how I created this blog. By the way, this the first post of my blog. In the future I'll write about infosec, the development  of my personal tools and HOWTOS of a variety of subjects .The website is entirely written in markdown, built with Jekyll and hosted/served by GitHub. 

## 1) Why Choosing Jekyll and Github ?
I wanted to create a blog for a long time to be able to share my findings/struggles/knowledge. Being familiar with markdown, I was looking for something that was able to serve MD pages, and that pretty easy to use. Since I just wanted to have a static website with a bunch of posts, Jekyll seamed like the perfect answer.

To remove the hassle of administrating and renting a web server, I searched and founded out that GitHub was able to serve Jekyll pages, free of charge, with all the benefits that came with it (redundancy/versioning/...). 

## 2) Installing Jekyll locally
I'm currently working on Debian 10 (Buster), so I'll install the [requirements] :
{% highlight bash %}sudo apt update && apt install ruby-full build-essential {% endhighlight %}

Once the requirements are installed, I can proceed with the [installation] Jekyll :
{% highlight bash %}gem install jekyll bundler {% endhighlight %}

Then I create the website in the current directory :
{% highlight bash %}
jekyll new amperclock.github.io
cd amperclock.github.io
{% endhighlight %}

Finally, once I'm in the directory, I can run the gem locally to serve the website:
{% highlight bash %}bundle exec jekyll serve {% endhighlight %}

## 3) Changing the standard theme
I was not a fan of the daymode of [Minima], the standard theme that come with Jekyll (yes, I'm carrying about my eyeballs). I choose to keep the same layout, but wanted it dark. Luckily, this feature was added to the version 3.0 of [Minima]. The issue is that, at the time of writing, it has not been released. I'll have to pull it directly from Github.

**NOTE : GITHUB PAGES DOESN'T SUPPORT CUSTOM THEMES YET. THIS CONFIGURATION IS USELESS IF YOU USE GITHUB PAGES. TO HAVE THE DARKMODE, I NEED TO WAIT FOR THE OFFICIAL RELEASE OF THE VERSION 3.0 OF MINIMA**

First, let's update the Minima's version:
{% highlight bash %}
vim Gemfile

change  gem "minima", "~> 2.0"
to      gem "minima", github: "jekyll/minima"
{% endhighlight %}

Then, update the _config of the website :
{% highlight bash %}
vim _config.yml

after   theme: minima
add     minima: 
            skin: dark
{% endhighlight %}


## 4) Hosting the blog on Github
Now that I've tested Jekyll locally, I'm ready to host it on Github to make it available to the internet.
I'm using the [Github Pages official documentation] to help me in the process.

I create a new repository for my website, with the name amperclock.github.io, according to the [Github Pages guidelines].
![Creating the repo](/assets/creating-repo.png)

Now I go to the the [GitHub Pages Dependency versions] and grab the version for `github-pages` (mine is 209) and `jekyll` (mine is 3.9.0).

I maje a few modifications to the Gemfile in order to have matching versions:
{% highlight bash %}
cd amperclock.github.io
vim Gemfile

change the line gem "jekyll", "~> 3.8.3"
to              gem "jekyll", "~> VERSION"
with VERSION being the version you grabed before (mine is 3.9.0)

under the line  # gem "github-pages", group: :jekyll_plugins
add             gem "github-pages", "~> VERSION", group: :jekyll_plugins
with VERSION being the version you grabed before (mine is 209)
{% endhighlight %}

Let's install locally the new versions:  
{% highlight bash %}bundle update && bundle install{% endhighlight %}
and test that everything still works:
{% highlight bash %}bundle exec jekyll serve{% endhighlight %}

In a terminal, let's initialize the repo:
{% highlight bash %}
git init amperclock.github.io
{% endhighlight %}

Then track every Jekyll's file, except the `vendor` directory:
{% highlight bash %}git add -- . ':!vendor'{% endhighlight %}

I Add the origin:
{% highlight bash %}git remote add origin https://github.com/Amperclock/amperclock.github.io.git{% endhighlight %}

Then commit and push the modifications:
{% highlight bash %}
git commit -m "First commit"
git push
{% endhighlight %}

If I need to change the parameters of the website, I can go on the repo, in the `Settings` tab, and scroll down to the `GitHub Pages section`

## 5) Accessing the blog
The blog is now accessible at https://amperclock.github.io/. 
NOTE : If you forced HTTPS in the settings, the website will obviously only be accessible in HTTPS.

[requirements]: https://jekyllrb.com/docs/installation/other-linux/
[installation]: https://jekyllrb.com/docs/installation/ubuntu/
[Minima]: https://github.com/jekyll/minima
[Github Pages official documentation]: https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll
[Github Pages guidelines]: https://docs.github.com/en/articles/about-github-pages#types-of-github-pages-sites
[GitHub Pages Dependency versions]: https://pages.github.com/versions/