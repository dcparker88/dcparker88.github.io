---
title:  "Escaping code in code block with Jekyll"
date:   2018-3-10 00:00:00 -0600
categories: liquid jekyll til
---
# Overview
Today, when writing a post, I ran in to an interesting error. I was getting Liquid errors while running my blog locally, from a code block in a post.

{% highlight text %}
{% raw %}
Liquid Warning: Liquid syntax error (line 113): Expected end_of_string but found id
Liquid Warning: Liquid syntax error (line 114): Expected end_of_string but found id
{% endraw %}
{% endhighlight %}

I was having a hard time figuring out how to tell Jekyll this was just code in a code block. Finally, I found out you can escape the code using `raw`:
{% highlight text %}
{% raw %}

{% raw %}

{% endraw %}
{% endhighlight %}

For example:

{% highlight text %}
{% raw %}

{% raw %}
Some code I want to escape
{\% endraw \%}

{% endraw %}
{% endhighlight %}


I still had to put the `\` in the block above to get it to escape the example, so I am still learning as well.
