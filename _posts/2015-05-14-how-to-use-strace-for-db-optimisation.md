---
modified: 2015-05-14
tags: [strace, optimisation, index, mysql]
title: "A lucky shot with strace to optimise database"
excerpt: "How can you get useful insights about running processes."
---
I received a task to improve our shipment tracking system.
A part of the task was to poll more frequently our partner site's remote statuses.
Basically I had to change the two times a day cadence to 2 times an hour.
We already had a cronjob to manage this task, but I was not really familiar with this sub-system.

I hoped a bit that maybe it is enough to reconfigure the job's crontab entry.
Anyway I wanted to play safely so checked first how long the given tasks run.
The result was really sad. The task worked on only four day's data, but ran for 75 minutes.
The math was simple 75 minutes was too much.

As far as I knew the job polls only the remote systems and saves the result in the database.

We did not have too many profiling tools installed to the related machine, so I chose to use the simplest available tool:
`strace`.

I ran the job in one terminal and started a `strace` in another.

The result was surpisingly conclusive, the character flow stopped for seconds at the following lines:
{% highlight c %}
write(6, "\17\1\0\0\3\n            select\n       "..., 275) = 275
{% endhighlight %}

Strace's `-s` switch helped to find the guilty query exactly.
It was a very simple one:

{% highlight sql %}
select field from table where remote_id=?
{% endhighlight %}

The table received its well deserved index in minutes and the next run of the job was only 14 minutes.
