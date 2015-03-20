---
layout: post
title: Process Performance Graph
excerpt: "Automatically generating CPU/Mem graphs in Linux"
tags: [linux, bash, script, performance, cpu, memory]
modified: 2015-03-20
comments: true
share: false
---

For my post about writing a [Go Packet Sniffer](/Go-Packet-Sniffer) I needed a way to quickly log
and graph the CPU/Memory usage of any process. Turns out that getting the CPU usage of a single
process is not as easy as it seems. After using some Google-fu, I found an answer on 
[Stackoverflow](http://stackoverflow.com/a/1424556) and an article that 
[graphs memory usage](http://brunogirin.blogspot.com/2010/09/memory-usage-graphs-with-ps-and-gnuplot.html) 
using `ps` and the gnuplot library. I cobbled together a bash script that combines and automates
everything together based on a process name and PID. To use, just install gnuplot and make sure the 
`gnuplot` command is in your path. Then run script like:

`./process_graph.sh Xorg`

You may also specify the PID manually if multiple processes use the same name, I.E. chrome:

`./process_graph.sh chrome 12127`

By default it will watch the process for 60 x 1 second samples and then output a graph as a png image. 
The timeout and number of samples can be changed by modifying the variables at the top of the script if
you want to watch the process for longer. This post is mainly so that I remember how to do this in the
future, but other people might have the same problem.

The full script is embedded in the gist below.

{% gist bisrael8191/39109bb6078c0ff54e33 %}