---
layout: post
comments: true
title:  "Getting specific chunks out of very large files"
date:   2014-10-22
categories: less vim "large files" chunk
---
Today I had to edit a very large (>50GiB) database dump. Vim wouldn't load it, at least not within an acceptable timeframe. I just had to remove the first 20 or so lines from it.

Now, there is probably a Vim plugin that enables you to load large files in chunks, but I didn't want to install anything on the machine I was on, so that limited my options.

After some googling I finally found a solution which uses `less` and it's pipe functionality. The relevant part in the man-page looks like this:

{% highlight man %}
| <m> shell-command
     <m> represents any mark letter.  Pipes a section of the input file to the given shell command.  The section
     of the file to be piped is between the first line on the current screen and the position marked by the let‐
     ter.  <m> may also be ^ or $ to indicate beginning or end of file respectively.  If <m> is  .  or  newline,
     the current screen is piped.
{% endhighlight %}

Now, because I only needed to remove a couple of lines from the top, it was actually really easy to use. Say I need to remove 20 lines, I'd do this:
{% highlight bash %}
less input
# next commands are in less
20j
|$
cat > output
{% endhighlight %}

The example above opened the file in less, which is _way_ faster than in Vim. Probably because less doesn't try to load the entire file in memory. Then it moves the cursor 20 lines downwards, and pipes everything from the cursor till the end of the file to `cat > output`.

<sub>... I'm still waiting on the file to finish writing to disk btw :D</sub>
