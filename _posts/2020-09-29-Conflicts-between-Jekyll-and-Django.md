---
layout: post
title: Conflicts between Jekyll and Django [Solved]
tag: quickfix Django Python Jekyll Liquid Blog
---

# Conflicts between Django and Jekyll

Just after publishing my second block post is encountered another issue with Jekyll.
Github pages failed to built and told me that the `url tag is not a Liquid tag`.

After some research I found out that `jekyll` uses `Liquid` tags for it's logic. If e.g. Jekyll wants to list all block posts it uses somewaht like this.
{% raw %}
```
{% for blog in blockposts %}
...
{% endfor %}
```
{% endraw %}

It obviously uses the same logic for dynamically rendering HTML pages as Django.
But Django has more tags than Lquid so some of them are unknown for Liquid and even the known ones would mess up the rendering

An easy solution is to wrap all the Django `HTML` codeblocks with `raw` tags.
They are not quite easy to display here since they are hidden in the rendered file but it is basically the same logic as allm the other tasks using `raw` and `endraw`
{% raw %}
```
{$ raw $} #replace $ with %

Some Django HTML code

{$ endraw$ } # replace $ with %
```
{% endraw %}

**Note:** Be in general carefull with using the % sign in Jekyll code blocks. 