---
title: "Talk @ Netdev 0x13: Improved System Call Batching for Network I/O" 
date: 2019-05-26T21:07:27+05:30
tags: ['talks', 'tech', 'linux']
---

My colleagues [Rahul Jadhav](http://blog.rjed.org/), Zhen Cao and me had our talk for [Netdev 0x13](https://netdevconf.info/0x13/) accepted. Our basic idea is reducing the CPU cycles consumed by amortizing the system call overhead of network I/O operations. 

At the time we started our work, [`io_uring`](https://lwn.net/Articles/776703/) was not yet a thing. With the benefit of hindsight however, it is clear that `io_uring` is the way forward.

Here's the video of Rahul delivering the talk: 

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/hJrXbqttJC4" frameborder="0" allowfullscreen></iframe></center>

And the slides:
<script async class="speakerdeck-embed" data-id="99baab134b764ae18553842caef1876a" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>
