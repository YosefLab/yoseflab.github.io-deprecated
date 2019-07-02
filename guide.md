---
layout: page
title: Contributing an article
permalink: /guide/
sitemap: false
---

If you're contributing an article, please read these instructions on how to write and post an article on this blog.

##Using Markdown

All articles are written in markdown with minimal formatting necessary. To a first approximation your article should simply be a plain text file. Here's the basic template your article should use:

~~~
---
layout:     post
title:      Using markdown
date:       2015-12-01 9:00:00
summary:    Learn to write a post in markdown
author:     Your name
visible:    False
---

Articles are written in markdown (kramdown). 
Please use minimal formatting for your article.

This is a single paragraph. It is separated by newlines. 
No need for html tags. 
A single line break does not start a new paragraph.

This is a another paragraph. 
You can place subsections as follows.

## Using maths

You can use math as you would in latex. Let $n>2$ be an integer.
Consider the equation:
\[
a^n + b^n = c^n
\]

Wiles proved the following theorem.

> **Theorem** (Fermat's Last Theorem).
> The above equation has no solution in the positive integers.

## Links and images

You can place a link like [this](http://wikipedia.org).

You place a picture similarly:

![Lander](https://upload.wikimedia.org/wikipedia/commons/thumb/4/45/Albert_Bierstadt_-_The_Rocky_Mountains%2C_Lander%27s_Peak.jpg/320px-Albert_Bierstadt_-_The_Rocky_Mountains%2C_Lander%27s_Peak.jpg)

## Emphasis and boldface

Use *this* for emphasis and **this** for boldface.

## You're all set.

There really isn't more you need to know.
~~~

The post would appear like [this](/guide/example/) on the web.
If you need more than what is in the above example, check out this 
[kramdown reference](http://kramdown.gettalong.org/quickref.html). 
You may also use any valid HTML tag in your article, but please try to avoid this.


##Submitting an article

The blog lives in this [github repository](https://github.com/YosefLab/yoseflab.github.io). If you are a regular contributor and your github account has admin access to the repository, you can add an article to the `/_posts` directory with your images added to the `/assets` directory. Please make sure your article is pointing to images in `/assets`.

For example the following HTML code is acceptable in the markdown document.
~~~
<p style="text-align:center;">
<img src="/assets/myimage.png" width="100%" alt="My image" />
</p>
~~~

If you don't have admin access, you can either create a [pull request](https://help.github.com/articles/creating-a-pull-request/) (assuming you're comfortable with git) or send any regular contributor the article in markdown. If so, please start from [this template](https://raw.githubusercontent.com/offconvex/offconvex.github.io/master/_posts/2015-12-01-template.md) and follow the above. An example of a more complex blog post is [here](https://raw.githubusercontent.com/offconvex/offconvex.github.io/master/_posts/2018-11-07-optimization-beyond-landscape.md).

If you would like to test the addition of your article you may install [Jekyll](https://jekyllrb.com) and use the command `jekyll serve` from inside the forked repository. You can then view a local build of the website by following the outputted instructions.
