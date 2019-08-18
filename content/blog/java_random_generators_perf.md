+++
title = "Performance characteristics of Random, ThreadLocalRandom and SplittableRandom"
date = "2019-08-18"
draft = false
author = "Alex Kachur"
categories = ["Performance", "Java", "Random"]
type = "post"
+++

Preface
--------------------------------------------
Recently I've got interesting task related to random data generation and writing this data on disk as fast as possible using Java. 
Unfortunately performance requirements were really tight: in peak application should support writing at ~3 Gb/s. 
Which means that it should generate random data at really high pace, encode it to proper charset and finally write to disk.
That's pretty much an exposition and motivation behind following measurements and why it was so important to dig in.

RandomStringUtils
--------------------------------------------
So it's needed to generate random string and write it to disk - obviously there should be a solution for such a common problem.
And indeed there is (at least) one - [RandomStringUtils](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/RandomStringUtils.html). 
It works just fine (despite bloating heap), until you need to generate file >1Gb which leads to hitting a lot of Java limits like maximum heap size and [maximum array size](https://stackoverflow.com/questions/3038392/do-java-arrays-have-a-maximum-size) (duh...)

DIY 
--------------------------------------------
At this point it became obvious that there is no free lunch and I need to write something by myself using pseudorandom number generator (PRNG).
In Java there are 3 options to choose from: [Random](https://docs.oracle.com/javase/7/docs/api/java/util/Random.html), [ThreadLocalRandom](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadLocalRandom.html) and [SplittableRandom](https://docs.oracle.com/javase/8/docs/api/java/util/SplittableRandom.html) so let's evaluate those options with JMH micro-benchmarks.

### Benchmarks
We'll use two simple scenarios for every class.
First one: generate 1 million random integers and sink them to a [Blackhole](http://javadox.com/org.openjdk.jmh/jmh-core/1.6.3/org/openjdk/jmh/infra/Blackhole.html) to avoid JVM optimizations
```java
for (int i = 0; i < 1000_000; i++)
			bh.consume(random.nextInt());
```
Second one: generate 1 million random integers with 1 byte bound to benefit from blazing fast ISO-8859-1 character encoding
```java
for (int i = 0; i < 1000_000; i++)
			bh.consume(random.nextInt(255));
```

Source code for this JMH project can be found here: https://bitbucket.org/conockrad/jmh-benchmarks.git

#### Test stands
Linux with Intel Core i3-7100U CPU @ 2.40GHz with OpenJDK 11
OS X  with Intel Core i7       CPU @ 2.2 GHz with Oracle JDK 8  
On both stands results were pretty much similar despite numbers were not quite the same

### Results
```bash
Benchmark                                             Mode    Cnt  Score    Error  Units
RandomGeneratorBenchmark.random                       thrpt    5   80.986 ±  9.801  ops/s
RandomGeneratorBenchmark.randomWithBounds             thrpt    5   78.548 ±  3.616  ops/s

RandomGeneratorBenchmark.threadLocalRandom            thrpt    5  221.914 ± 23.123  ops/s
RandomGeneratorBenchmark.threadLocalRandomWithBounds  thrpt    5  112.041 ± 15.624  ops/s

RandomGeneratorBenchmark.splittableRandom             thrpt    5  251.625 ± 12.932  ops/s
RandomGeneratorBenchmark.splittableRandomWithBounds   thrpt    5  129.244 ±  9.871  ops/s
```
### Analysis
There is a great [article](http://alvaro-videla.com/2016/10/inside-java-s-threadlocalrandom.html) by Alvaro Videla on internal implementation of ThreadLocalRandom and SplittableRandom. So if you're interested in deep dive - go ahead, it's worth it!
#### Random
Being synchronized and almost deprecated no wonder that java.util.Random is the slowest one and no further discussion is needed.

<img src="./random.svg" width="100%"/>
#### ThreadLocalRandom
So fun part starts here. As you can see bounded version two time slower than method without any restrictions: 

```bash
RandomGeneratorBenchmark.threadLocalRandom            thrpt    5  251.625 ± 12.932  ops/s
RandomGeneratorBenchmark.threadLocalRandomWithBounds  thrpt    5  129.244 ±  9.871  ops/s
```
The same applies to SplittableRandom since both implementations share the same code:  
<img src="./diff.png" width="100%"/>

Most probable version is that time spent in "rejecting over-represented candidates" section but it's hard to find any evidence since async-profiler couldn't provide more information than following:

ThreadLocalRandom:
<img src="./threadlocal.svg" width="100%"/>

ThreadLocalRandom with bounds:
<img src="./threadlocal_bounded.svg" width="100%"/>


#### SplittableRandom
SplittableRandom runs a bit faster but overall behavior is similar to ThreadLocalRandom
SplittableRandom:
<img src="./splt_random.svg" width="100%"/>

SplittableRandom with bounds:
<img src="./splt_random_bounded.svg" width="100%"/>

#### Workaround
So knowing internal structure of limiter method I modified to fit my needs:
```java
public int nextInt(Random random, int bound) {
		return random.nextInt() & (bound - 1);
}
```

And results with such modification pretty satisfying:
Benchmark                                              Mode  Cnt    Score    Error  Units
RandomGeneratorBenchmark.splittableRandomWithBounds   thrpt    5  223.632 ± 26.459  ops/s
RandomGeneratorBenchmark.threadLocalRandomWithBounds  thrpt    5  226.788 ± 33.923  ops/s

It's not a silver bullet since it cuts logic which fixes overrepresntation of some values but considering my case it just works

### Conclusion
1. java.util.Random is outdated and ThreadLocalRandom or SplittableRandom should be PRNG class of a choice for most cases
2. It appeared that providing bound option one may cut performance of a random generator in half wich is barely convenient
3. With understanding of internal structure it's possible to overcome of platform limitations