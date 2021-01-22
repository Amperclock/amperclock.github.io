---
layout: post
title:  "Creating the blog with Jekyll"
date:   2021-01-22 13:30:34 +0100
categories: jekyll Creation
---
Sooo, this is the first post of this blog. In the future I'll write about infosec, the development  of my personal tools and HOWTOS of a variety of subjects .The website is entirely written in markdown, built with Jekyll and hosted/served by GitHub. 

## 1) Why Choosing Jekyll and Github ?
I wanted to create a blog for a long time to be able to share my findings/struggles/knowledge. Being familiar with markdown, I was looking for something that was able to serve MD pages, and that pretty easy to use. Since I just wanted to have a static website with a bunch of posts, Jekyll seamed like the perfect answer.

To remove the hassle of administrating and renting a web server, I searched and founded out that GitHub was able to serve Jekyll pages, free of charge, with all the benefits that came with it (redundancy/versioning/...). 

## 2) Installing Jekyll locally
I'm currently working on Debian 10 (Buster), so I'll install the [requirements] :
{% highlight bash %}sudo apt update && apt install ruby-full build-essential {% endhighlight %}

Once the requirements are installed, I can proceed with the [installation] Jekyll :
{% highlight bash %}gem install jekyll bundler {% endhighlight %}

Then create the website in the current directory :
{% highlight bash %}
jekyll new amperclock.github.io
cd amperclock.github.io
{% endhighlight %}

Finally, once I'm in the directory, I can run the gem locally to serve the website:
{% highlight bash %}bundle exec jekyll serve {% endhighlight %}

## 3) Chaging the standard theme
I was not a fan of the daymode of [Minima], the standard theme that come with Jekyll (yes, I'm carrying about my eyeballs). I choose to keep the same layout, but wanted it dark. Luckily, this feature was added to the version 3.0 of [Minima]. The issue is that, at the time of writing, it has not been released. I'll have to pull it directly from Github. 

First, let's update the Minima's version:
{% highlight bash %}
vim Gemfile

change  gem "minima", "~> 2.0"
to      gem "minima", git: "https://github.com/jekyll/minima"
{% endhighlight %}

Update the site :
{% highlight bash %}bundle update && bundle install {% endhighlight %}

And configure the dark theme in the `_config.yml` file :
{% highlight bash %}
vim _config.yml

after   theme: minima
add     minima: 
            skin: dark
{% endhighlight %}


## 4) Hosting the blog on Github
Now that I've tested Jekyll locally, I'm ready to host it on Github to make it available on the web.
I'm using the [Github Pages official documentation] to help me in the process.

I create a new repository for my website, with the name amperclock.github.io, according to the [Github Pages guidelines].
![Creating the repo](/assets/creating-repo.png)

Now, in a terminal, let's initialize the repo:
{% highlight bash %}
git init amperclock.github.io
{% endhighlight %}

Go to the the [GitHub Pages Dependency versions] and grab the version for `github-pages` (mine is 209) and `jekyll` (mine is 3.9.0).

Make a few modifications to the Gemfile:
{% highlight bash %}
cd amperclock.github.io
vim Gemfile

change the line gem "jekyll", "~> 3.8.3"
to              gem "jekyll", "~> VERSION"
with VERSION being the version you grab before (mine is 3.9.0)

under the line  # gem "github-pages", group: :jekyll_plugins
add             gem "github-pages", "~> VERSION", group: :jekyll_plugins
with VERSION being the version you grab before (mine is 209)
{% endhighlight %}

Let's install the new versions:  
{% highlight bash %}bundle update && bundle install{% endhighlight %}
and test that everything still works:
{% highlight bash %}bundle exec jekyll serve{% endhighlight %}


[requirements]: https://jekyllrb.com/docs/installation/other-linux/
[installation]: https://jekyllrb.com/docs/installation/ubuntu/
[Minima]: https://github.com/jekyll/minima
[Github Pages official documentation]: https://docs.github.com/en/github/working-with-github-pages/creating-a-github-pages-site-with-jekyll
[Github Pages guidelines]: https://docs.github.com/en/articles/about-github-pages#types-of-github-pages-sites
[GitHub Pages Dependency versions]: https://pages.github.com/versions/