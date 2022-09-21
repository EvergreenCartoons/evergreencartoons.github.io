---
layout: post
title: "Editing logs surgically with ed"
---

Recently while working on the cleanup script for [SenselessViolence][SenselessViolence], I was trying to come up with a cleaner way to zap log entries from a logfile, without just deleting the whole file.

It also had to work in-place, without copying/editing/rewriting the logfile.

Anyway, with some use of `printf` and `ed`, it seems this is possible.

The following snippet will erase any log file entry containing the string "python-requests".

{% highlight bash %}
printf '%s\n' 'g/python-requests/d' w q | ed -s /var/log/nginx.log
{% endhighlight %}

So far, this seems to work fine against the FreeBSD target (pfSense) for zapping the suspect log entries without just rm'ing the logfile.

I guess next step is testing it against ssh auth logs?

Writeup on the design choices in SenselessViolence to follow.

[SenselessViolence]: https://github.com/EvergreenCartoons/SenselessViolence
