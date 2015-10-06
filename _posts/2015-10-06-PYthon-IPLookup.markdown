---
layout: post
title:  "Python, BGP RIB Table and IP Lookup
date:   2015-10-06
author: Abhijit Gadgil
categories: Python
tags: python, numpy, iplookup
---


Recently, I was looking at the RadixIPLookup element in [Click](http://www.read.cs.ucla.edu/click/elements) about an implementation of a routing table, which could do very fast lookup on a routing table of about 167K entries. I was looking at another fast IP lookup for a related problem at work - though not on such a large table size and the problem was that of a fixed match, so a simple hash table should suffice. Interestingly a while back I'd looked at some of the datasets from [caida.org](http://data.caida.org/datasets/) while I was looking around about net neutrality. Specifically I was looking at whether there's any data that could tell me what percentage of IP addresses are local to a country (say India). 

So this posted an interesting problem - how to implement a longest prefix match lookup for IP. So I wrote a bunch of utils in python to play around with the data that I had, specifically trying to find out what percentage of IPs belong to India and/or outside. It turned out more challenging than I initially thought. But eventually I'd something working. 

Here was the plan - use the [routeviews RIB data](http://archive.routeviews.org/bgpdata/) to parse the MRT RIB data to know about the routing prefixes and the AS where there originate. Use this information and AS information from the caida.org data above to map AS to country and thus transitively map prefixes to country. All we'd then need is a routing table that implements a longest prefix match and then we'd be able to map a destination IP address to a country. 
A while back I'd written a [post](https://oltsm.wordpress.com/2015/04/18/almost-everything-about-net-neutrality/) about Net neutrality, especially after a heated debate about it. Also, I have been thinking why is Internet bandwidth is so expensive here, the reasons are most likely going to be economic, but a common argument I have heard many times in the past is - the cost of international bandwidth is very high. I was thinking, if there's indeed a way to actually figure out what percentage of traffic is International and what percentage of traffic stays in India (and hence should not be charged atrociously). Interestingly, just around then I came to know about [caida.org](http://data.caida.org/datasets/), which has got many interesting datasets. I thought it might be worthwhile to find some interesting facts about data from these datasets.

So here was the plan - look at the routing data, more specifically RIB data and parse it. The data is in [MRT](http://tools.ietf.org/rfc/rfc6396.txt) format. I wrote a small python utility that dumps the RIB TABLE_DUMP_V2 data and gives a list of RIB entries. This data looks something like following.
