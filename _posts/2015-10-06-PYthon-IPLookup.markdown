---
layout: post
title:  Python, BGP RIB Table and IP Lookup
date:   2015-10-06
author: Abhijit Gadgil
categories: Python
tags: python, numpy, iplookup
---

Recently, I was looking at the RadixIPLookup element in [Click](http://www.read.cs.ucla.edu/click/elements) about an implementation of a routing table, which could do very fast lookup on a routing table of about 167K entries. I was looking at another fast IP lookup for a related problem at work - though not on such a large table size and the problem was that of a fixed match, so a simple hash table should suffice. Interestingly a while back I'd looked at some of the datasets from [caida.org](http://data.caida.org/datasets/) while I was looking around about net neutrality. Specifically I was looking at whether there's any data that could tell me what percentage of IP addresses are local to a country (say India). 

So this posted an interesting problem - how to implement a longest prefix match lookup for IP. So I wrote a bunch of utils in python to play around with the data that I had, specifically trying to find out what percentage of IPs belong to India and/or outside. It turned out more challenging than I initially thought. But eventually I'd something working. 

Here was the plan - use the [routeviews RIB data](http://archive.routeviews.org/bgpdata/) to parse the MRT RIB data to know about the routing prefixes and the AS where there originate. Use this information and AS information from the caida.org data above to map AS to country and thus transitively map prefixes to country. All we'd then need is a routing table that implements a longest prefix match and then we'd be able to map a destination IP address to a country. 

[Here](http://github.com/gabhijit/ipgiri) is the code for everything, the MRT file parser and IP routing table. A few words about the IPv4 routing table as that was the most interesting part (especially implementing it in Python). Some of the ideas are based from this [thesis](http://klamath.stanford.edu/~pankaj/phd.html). So it's basically a multi-level table each level looking at some bits in the IP adress. So the table is structured as follows

 Level 0 - 64K entries (first 16 bits)
 Level 1 - 256 entries (next 8 bits) 
 Level 2 - 16 entries  (next 4 bits)
 Level 3 - 16 entries  (next 4 bits)

So When we have a /16 prefix in a routing table - we'd generate only one entry in the routing table corresponding to the index in the routing table for the first 16 bits of the IP address. If we have less than 16 bits in prefix, we'd populate all entries that start at the base offset of a prefix and continue for the rmaining bits. eg. a prefix like 132.16/12 (1000 0100 0001 0000 / 12) would have all entries from (132*256+16) to (132*256+31) populated. Similarly a /24 prefix will have it's first 16 bit's matched to a prefix in the Level 0 table and next 8 bit would corresponding to an entry inn Level 1 table and so on. This leads to a certain amount of wastage of space, as there'd be duplicate entries if the prefixes do not match the exact bits for a given level. Based on the data - as the table size grows this leads to about 10 entries being allocated for every prefix (a considerable waste of space), but the implementation of table is 'very simple' and worth it for a simple problem at our hand. So each entry looked like following 

{% highlight python %} 
class RouteEntry:
    def __init__(self, pre_len, final, output_idx):
        """Prefix Length of this Entry. Whether this entry is final or not and
        output index for this entry (only valid if this entry is final)."""
        self.prefix_len = pre_len
        self.final = final
        self.output_idx = output_idx
        self.children = None

{% endhighlight %}
The `children` property is set to None upon initialization. Whenever we add next level prefix(es) entry(ies), this property is set to a List of another `RouteEntry` objects like this. The interesting part of this is - though the entry looked very simple it was occupying approximately 400 bytes! This rendered it clearly unusable for a fairly recent routing table size of 587K entries (approximately 5million entries in table still substantially lesss than 2 billion entries for 2^32 IP addresses), as the memory required was about 2.5G, something I couldn't work with on my Virtualbox VM with 1G memory. Even if I gave it 3G of memory, populating this table was extremely 'slow' eg. it took about 15 minutes to populate about 50K entries so it'd have taken nearly 2 hours to populate entire table (certainly not worth)! I was looking for choices - So I first tried using a packed `struct` for the entries as most of the entries could be covered by integers only few bits wide. So yes - from that angle as well the table was a total waste. The `struct` approach didn't do any better, but instead actually became a bit slower and started consuming slightly more memory. What I was looking at is something that could use about 80M of memory (16 byes per entry times approximately 5million entries). The next thing that I explored was `numpy`. Interestingly `numpy` provides support for array of records (called `numpy.dtype`), which can be tightly fit into memory and easily accessed. So the `numpy` routing table entry looked something like following - 

{% highlight python %}
RouteEntryNP = np.dtype([('final', 'u1'), ('prefix_len', 'u1'),
                            ('output_idx', '>u4'), ('children', 'O')])
{% endhighlight %}

`numpy.dtype` has got an excellent documentation and lot of it is self explanatory. So the 'raw' data it uses is about 14 bytes (actually I could make this a little more cache friendly by adding two Padding bytes and make this a 16 byte entry). When this routing table ntry was used, we could read the entire table in about 25 minutes and eventually only 160M of space was occupied (it's still a mystery to me - the space occupation still looks more than what one can do with simple back of the envelope calcuation). This was something I could test with some 'known' IP addresses and their ASes (eg. we know 8.8.8.8 is google or geoIP lookup showed my IP belonged to my ISP). The results were accurate, so we can say that the whole scheme works atleast at the first level. So something we could experiment with. 

The next option I was looking at was - can I actually somehow 'shortern' the table population time? 23 minutes is still very high, you could make your own tea, have it and still the table would be populating. Fortunately, `numpy` provides two excellent functions `save` (actually there are a few varieties of `save`) and `load` that allow storing and loading array data to a file. So first time when I read the table assuming it's not going to be very 'dynamic', I could just save it to a file and read it whenever I wanted to do any lookups. What I found very interesting about this `numpy` feature is, most of the arrays had an Object reference to a Python Object. Still the `store` and `load` were totally seamless and very fast (about 5-10 seconds for a load). I did face a couple of issues with `store` especially, when I was trying to store as a compressed. The memory usage of the Python process shot quite substantially and the process came to a halt eventually when the mem usage approached my VM memory limit. I actually had to restart my VM with higher memory for it to finish. Another issue that I observed was - when the file was loaded, the memory usage shot up and nearly stayed there for the Python process. Though it didn't affect lookup speed the memory usage was rather large (about 950 MB). This is a piece of puzzle I am yet to resolve. But I'd got something working starting with a 'vague' idea about how to go about doing it and that itself was very satisfying, along the way I learnt a few things about python `struct` and `namedtuple` as a bonus. A few takeaways are - 
 - having a basic working code of something is not very difficult
 - doing something non-trivial (working with actual real world data) is considerably more challenging than one may foresee
 - `numpy` is just a great package. I should surely learn more about it.

 
