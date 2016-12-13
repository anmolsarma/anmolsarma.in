
+++
date = "2015-04-04T18:40:03.888Z"
draft = false
title = """Lazy Performance Comparison of WordPress and Ghost"""
slug = "lazy-performance-comparison-of-wordpress-and-ghost"
tags = ['tech']
banner = "/images/2015/04/Ghost_WordPress_Load.png"
aliases = ['/lazy-performance-comparison-of-wordpress-and-ghost/']
+++

So, I switched to a new blogging platform. Again. After trying out a static approach, I moved back to a WordPress when I stumbled across [OpenShift](https://www.openshift.com/) (*And their fantastic [free plan](https://www.openshift.com/products/pricing/plan-comparison)*). That was well over two years ago. In the meantime, I hardly blogged and didn't really run into any issues but the blog did feel a tad sluggish.

Fast-forward to yesterday when I had a sudden impulse to try something new. Ghost looked interesting so, I spun up an install using OpenShift's [Ghost Quick-Start](https://github.com/openshift-quickstart/openshift-ghost-quickstart) and proceeded to [import data from my WordPress](https://ghostforbeginners.com/how-to-transfer-blog-posts-from-wordpress-to-ghost/) blog which turned out to be pretty painless.

Given the same resources (*Small Gear*) and same amount tweaking (i.e. none, using OpenShift's quickstart installation defaults and using the default themes with no plugins), Ghost certainly *felt* a lot faster but how fast was it really? To answer that (*And because I apparently have nothing better to do on a Friday night*), I started a simultaneous stress test on WordPress and Ghost using [BlazeMeter](http://blazemeter.com/) with a maximum of 50 users. The answer, as turns out is a **helluva lot faster!**

<style type="text/css">
table{
border-collapse: collapse;
border: 1px solid black;
width: 100%;
}
table td{
border: 1px solid black;
padding: 5px;
}
</style>
<table>
<tr><td> </td>
<td align="right">Avg. Latency</td>
<td align="right">Avg. Response Time</td>
<td align="right">90%</td>
<td align="right">95%</td>
<td align="right">99%</td>
<td align="right">Min</td>
<td align="right">Max</td>
</tr>
<tr><td>Ghost</td>
<td align="right">255.05</td>
<td align="right">752.28</td>
<td align="right">739</td>
<td align="right">793</td>
<td align="right">4647</td>
<td align="right">602</td>
<td align="right">7240</td>
</tr>
<tr><td>Wordpress</td>
<td align="right">12129.55</td>
<td align="right">21238.06</td>
<td align="right">33641</td>
<td align="right">37725</td>
<td align="right">52178</td>
<td align="right">2273</td>
<td align="right">67093</td>
</tr></table>


On average, Ghost's response is **28 times faster** and its latency is **47 times lower**. Looking at how the two respond to load is interesting:

![](/images/2015/04/Ghost-wordpress-responsetime.png)

![](/images/2015/04/Ghost_Wordpress_latency.png)


Ghost doesn't even seem to break a sweat while WordPress starts panting quite early into the run. The longest it took for Ghost to respond was **7240 ms** which is almost a whole order of a magnitude faster than Wondpress's **67093 ms**. The shortest time for Ghost is **602 ms**, a comfortable **3.77 times** better than **2273 ms** it took WordPress.

Basically Ghost crushes WordPress in this admittedly lazy and unscientific test. It is pretty similar to what others have reported. If your personal WordPress setup is struggling to keep up with traffic, you should at least consider Ghost. It doesn't have anywhere near the number of plugins or themes as WordPress, it clearly is not as mature (*It's still in version 0.5.10*) but boy is it fast! As for the writing flow, I can only say its different. I'm not entirely sure if it is better than what we have been used to (*No spell check*).

I for one am quite happy I tried (*And switched to*) Ghost. And I got a blog post out it to boot!



