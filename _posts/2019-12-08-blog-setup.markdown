---
layout: post
title:  "Setting Up This Blog"
date:   2019-12-08
---

<p class="intro"><span class="dropcap">I</span>t has been quite a while since I wanted to setup a personal vanity blog. Earlier I had tried setting this up using <a href="https://blog.getpelican.com/">Pelican</a>, but wasn't quite impressed by the themes available then, so thought of trying something else later. After 3.5 years, I am finally setting it up again using <a href="https://jekyllrb.com/">Jekyll</a> this time.</p>

Jekyll is pretty straight forward to setup. The installation steps are given at <a href="https://jekyllrb.com/docs/">Jekyll docs</a>. The theme that I used is called <a href="https://github.com/brianmaierjr/long-haul">Long Haul</a>. After cloning the theme, I just need to do the following to generate the static site.

{% highlight bash %}
> jekyll b 
{% endhighlight %}

<p>To test it locally, I just need to do this and it would start a local server at port 4000</p>
{% highlight bash %}
> bundle exec jekyll serve
{% endhighlight %}

The static website is generated at `_site/` directory, which I can easily copy to my gh-pages repo and it's done !
