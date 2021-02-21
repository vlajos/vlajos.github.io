---
modified: 2015-08-17
tags: [cdn, akamai, rackspace, nginx, iftop, apache, netstat, newrelic, geckoboard, apdex, postmortem]
title: "Orchestrating a high traffic static page after the last minute"
excerpt: "A presumably calm Monday with some trouble and network debugging."
header:
  overlay_image: /images/windsor.jpg
---

Some background:

We have launched one of our new pages a couple of days ago:
[Perceptions of Perfection](https://onlinedoctor.superdrug.com/perceptions-of-perfection/)

The backend team was not very involved in the development of this new page because it seemed to be relatively simple and frontend-only.
We knew that it meant to be viral but we had no clue that this would be viral...

When I arrived Monday morning at the office, they were waiting for me with the news that something was not OK, the whole system was slow.
I checked my emails and saw that there were a couple of New Relic alerts too.

I logged in to our frontend web server and realised that something was really wrong. Even the ssh connections lagged.
The system load and `vmstat` did not seem to be too bad and there were not any interesting error log entries.
I checked the site. Both our static and generated pages were very slow.
Which meant in this case more than 20-30 seconds of loading time.

I checked our Geckoboard and saw that the apdex scores were very wrong. I assumed that maybe a bot attempting to place automated
orders or something similar. I checked the server logs whether there were any suspiciously high traffic ip addresses but found none.
My colleagues had let me know that the static pages were also very slow. Which meant the bot assumption was false...

I continued the investigation with the [server statuses](http://httpd.apache.org/docs/2.4/mod/mod_status.html).
This is one of apache's most useful and (I think) less-known features.
And we saw our whole system on top of its head...
Most of the processes were in state "G". It can be translated to "Gracefully finishing". But what did this mean exactly?
It was not very obvious and it was not 100% sure but it looked like our proxy webserver failed to handle the connections and
dropped them before draining their content properly.
So something had to be wrong a step before the apaches. I realised that this had to be a nginx problem.

We also noticed that most of our requests were against the same page.
It was a relatively simple AJAX endpoint that was used to display the expected delivery modes for our sites.
After a few minutes thinking I realised that this page's result changed only once a day...
We could cache it and we could temporarily replace the whole call with the static result.
We chose the second option as a quick fix and one of my colleagues started to implement a proper caching logic.
This was our first victory. The distribution of the apache requests looked instantly closer to normal than before and the site
gained some responsiveness. But it was still far from perfect.

So we returned to our previous point: the cause of "G"s. I wanted to verify what the situation was around our nginx.
After some typos and troubles, I enabled the [equivalent of mod_status](https://rtcamp.com/tutorials/nginx/status-page/)
on our nginx server using this [page](http://www.cyberciti.biz/faq/nginx-enable-and-see-current-status-page/).

I checked the numbers and saw that we had 2000 connections and most of them were waiting... Hmm.
2000 is very close to 2048.
I opened our nginx config and realised that we had only 1 `worker_process`... khm. Our server had 4 cpu cores (although virtuals).
And the `worker_connections` value was 2048...

I did a quick nginx tuning, increased the number of worker processes to 8, and enabled `multi_accept`. So with these new values:

{% highlight raw %}
worker_processes  8;
multi_accept on;
{% endhighlight %}

the whole system suddenly became more responsive. This was our second victory.
Unfortunately, it did not help much longer than the previous one...
We checked it with firebug and the connection seemed to be OK, but the page was still not stable.
It was still lagging and loading very slowly.

However, the site became a bit quicker but a couple of pages gave `502 bad gateway`. Classic. Something was wrong with the apaches.
With removing the block, enabling more nginx throughput, apache became the bottleneck again. But why?
Mod_status gave us the answer. Most of the connections were blocked. A simple apache restart fixed this bad gateway issue.

The whole system became slightly better but it was still far from optimal.
The nginx stats looked OK right now, but something was even wrong.
My next ideas were tcpdump and netstat. Maybe something was wrong around SYN handling? TCP backlog issue?
`netstat -s`, `netstat -taupn |grep -c SYN`

I verified quickly but there were only a couple of connections in SYN_RECV. Fortunately `netstat` gave us more clues.

The number of waiting connections was high. After some reading, it was obvious that they were only keepalive connections,
anyway decreasing anything which could use resources seemed to be a good idea, so we decreased the `keepalive_timeout` to 45 from 75.
The number decreased as well, but the responsiveness did not change.

So return back to netstat. Listing, searching, filtering and grouping the netstat results and watching it for a couple of minutes, I
realised that the Send-Q values were high, 30-40 for hundreds of connections.
It looked like something saturated. After a quick google search, I found [iftop](http://www.ex-parrot.com/pdw/iftop/),
a very neat and simple tool to monitor our network
interface's usage. Without `-n` (disable hostname lookup) it is mostly useless as most of the network tools, but with `-n` it showed us that
we had 100Mb traffic... Hmm... I wanted to verify whether our interface was 100Mb or maybe 1Gb but I did not find any working tool.
So we assumed it was 100Mb... It was not a 100% safe bet but it seemed to be easier to fix it than verify properly.
(This was the point when I cursed the third time who installed our www machine and forgot to enable `munin`... and myself for not having fixed this issue yet...)
Anyway, with the knowledge that our network was saturated, I wanted to verify why.

I checked our apache logs and the page above. I did not believe the results. There were requests with 1.8M result...
I asked the frontend team to optimise those images quickly and they reduced their size to the third of the original.

In the meantime, I started to bootstrap a rackspace CDN service. It was surprisingly easy.
I had never worked with CDN services before.
In a nutshell, they work as a reverse proxy. You add a name and the "original" server and it starts to serve the original server's
content on the new name. And you can add rules for caching. It would be perfect if the caching worked...
Anyway, the frontend team changed the URLs quickly to the CDN's and I started to debug why it was not caching.

With adding a simple GET parameter to an image request it was easy to separate
the test requests from the other storming traffic. `Wget`'s `--debug` showed us the certificate... Hmm, Akamai... Useful information.
I tried different caching rules but I got the same results.
None of them helped. Finally, I added a rule to cache everything for a day.
(Later I realised that the default cache settings were perfect. Images are still cached for a day.)
I compared the requests, assuming maybe the CDN's caching behaviour was influenced somehow by the original request's cache controlling
headers.
There were two interesting things: the original requests could be cached for 1 week and there was a `Vary: Accept-Encoding`.
What was this?
As a conclusion, it was obvious that the caching rule cannot be inherited by the original server's rules.
1 week could not be interpreted as nothing.
But what is this `Vary:` ?
I remembered it should help cache servers to differentiate between requests and cache them separately based on the headers defined by `Vary`.
This should not have been a problem.
But why it was there? Anyway, simplicity sounded better and because this was our only remaining clue, I googled for it.
[Success](https://www.drupal.org/node/2213429)! Or at least one more hint.
Akamai did not like that `Vary:` header in other cases. Maybe we had the same problem.
Anyway, we did not need these headers for our images. They could not be compressed anyway.
But our whole directory had this `Vary:` header. I narrowed down its scope with a `location` to match only to the javascript and css files.
And finally, the CDN started to serve clients from its cache and traffic dropped from 100Mbit to 5-8Mbit on the origin server.
The site became fully responsive. We had few thousands of visitors at the same time.
Happy End.

Until the end of the day, we had 120k shares, 9k likes, and 4k tweets.
