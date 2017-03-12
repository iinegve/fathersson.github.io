---
title: "HBase's Compaction Mechanism"
date: 2014-07-11
---

Here is the use-case we've faced once developing storage platform, which gives a lot of insights of how HBase’s compaction mechanism works and can be twicked. Thorough description of Minor compaction properties can be found [on the official site](http://hbase.apache.org/book/regions.arch.html#compaction).

### Requirements
- Daily import process. Each new day we import new chunk of data collected during previous day. Each day we have roughly equal amount of data (60+ Gb in gzip).
- TTL (time-to-live) for each day of data is 3 months.
- It requires to have the quickest access to the most recent data (for analysis, etc).

It is useful for us to utilize bulkload import instead of bother ourselves with standard Put / Delete approach.

### Invention of inevitable

By default, having major compaction mechanism turned on, once in a while all HFiles we have in a region are compacted into a huge one HFile. Since it requires to have quick access to last week of HFiles, then it’s probably a good idea to have several HFiles somehow distributed in terms of size. For example we might have something like this: the older file, the bigger size.

![pic 1]({{ site.url }}/assets/2014-07-11-hbase-compaction-1.png "Size of HFiles in days. There are 6 files in picture of different size")

Meaning is that for given 3 months (or 12 weeks) of data we’d have HFiles in different sizes. Oldest one would contains 6 weeks of data, another one would have 3 weeks, etc, until couple of newest would contains only couple of days or even one day of data. Such a distribution would help us utilize HBase’s intrinsic time-range filter mechanism, which could significantly reduce number of HFiles for reading.

Ok, let’s recall Minor Compaction parameters to check if it’s possible to set distribution similar to the one on the picture. So, as documentation stated there are only five parameter that influence Minor compaction:

- hbase.store.compaction.ratio Ratio
- hbase.hstore.compaction.min (files, default 2).
- hbase.hstore.compaction.max (files, default 10).
- hbase.hstore.compaction.min.size (bytes, default 128 mb).
- hbase.hstore.compaction.max.size (bytes, default Long.MAX_VALUE).

I’ve pulled out default HFiles selection algorithm from `o.a.hadoop.hbase.regionserver.Store` to get some insight of how different parameters influence the result. Here are some examples.

Since each day my data is roughly equal in size to the other days, for the sake of simplicity, I’ve changed size parameter measurement - I choose to use days instead of bytes.

~~~
    compaction.ratio = 1.2
    compaction.min = 2
    compaction.max = 2
    compaction.min.size = 0
    compaction.max.size = 16 (days, not bytes)
~~~
And here is the result with these parameters
~~~
    167        {25}  {16}  {16}  {16}  {16}   {4}   {1}   {1}   {1}   {1}
    167  m     {25}  {16}  {16}  {16}  {16}   {4}   {2}   {1}   {1}
    168        {25}  {16}  {16}  {16}  {16}   {4}   {2}   {1}   {1}   {1}
    168  m  M  {18}  {16}  {16}  {16}  {16}   {4}   {2}   {2}   {1}
    169        {18}  {16}  {16}  {16}  {16}   {4}   {2}   {2}   {1}   {1}
~~~
1. Major compaction occurs once in a week. Minor compaction occurs on each step.
2. Here is output twice for each step - first time without compaction, second time with compaction (including Major if that has place).
3. Keep in mind that TTL is set to ~90 days.

So, with these configuration we have 10 HFiles, the biggest HFile contains no more than 31 days of data and we’ve conquered our requirements - the newest HFiles would be read much faster than all others. But here is the important thing - HFiles in the middle would be read with equally same speed and only oldest data would be the slowest.

**Ok, let’s try different parameters.**

~~~
    compaction.ratio = 1.2
    compaction.min = 2
    compaction.max = 4
    compaction.min.size = 0
    compaction.max.size = 16
~~~
~~~
    105  m  M  {84}   {4}   {2}   {1}
    106        {84}   {4}   {2}   {1}   {1}
    106  m     {84}   {8}
    107        {84}   {8}   {1}
    107        {84}   {8}   {1}
    108        {84}   {8}   {1}   {1}
~~~
In the worst day we would have just 3 HFiles. Despite the fact that newest week would be the fastest for processing, everything else would be read slowly because of its huge size.

**One more example**

~~~
    compaction.ratio = 1.2
    compaction.min = 2
    compaction.max = 2
    compaction.min.size = 8
    compaction.max.size = 16
~~~
^
~~~
    945        {19}  {17}  {17}  {17}  {17}   {9}   {1}   {1}
    945  m  M  {29}  {17}  {17}  {17}   {9}   {1}   {1}
    946        {29}  {17}  {17}  {17}   {9}   {1}   {1}   {1}
    946  m     {29}  {17}  {17}  {17}   {9}   {2}   {1}
~~~

The picture hasn’t changed much from first example. couple of newest days would be the fastest, and everything else with roughly same access time as it was before.

Comparing different set of parameters we choose the first one. It gave us the better distribution, despite the number of files - it is slightly worse than in other cases, but we have faster access to newest week. And it’s even better, because if somebody would like analyse first 3 days only, it still be better than in other cases.

### Welcome to Reality.

Here is the result of Minor compaction working against the data.

One more something.
One more something.

... to be continued ...