Ever since the [first set of Warp benchmarks](http://docs.yesodweb.com/blog/announcing-warp) went live, people have been curious to see how Warp compares against non-Haskell web servers. Instead of simply throwing together some benchmarks on our local systems, Greg Weber, Matt Brown and I (Michael Snoyman) decided to create a fully reproducible set of benchmarks. For those of you dying of curiosity, here are the results, the explanation follows.

### Pong benchmark, extra large instance, requests/second
<img src="/static/benchmarks/2011-03-17/extra-large.png">

All of the code that went into this benchmark is available in a [Github repo](https://github.com/snoyberg/benchmarks). The file [setup.sh](https://github.com/snoyberg/benchmarks/blob/master/setup.sh) is actually a shell script which will install all dependencies for the benchmarks, pull the repo and run the tests. The script uses apt-get, and in theory may work for any Debian derivitive, but has only been tested using the Ubuntu <abbr title="Amazon Machine Instance">AMI</abbr>.

The test results above were run against an Amazon EC2 extra large instance. I ran the tests on a micro instance as well. However, due to lower RAM, some of the non-Haskell servers fared very poorly, and I decided to only publish the extra large instance results for now. As usual, results will vary based on the setup you run it on, but so far in every setting Warp comes in at least twice as fast as the next contender (excluding Yesod... I'll explain that in a second).

Now the caveats:

* I'm not an expert on Ruby, Python, PHP, node.js or Java. (I'm not even sure if I'm an expert in Haskell.) We've all tried our best to set up the most fair comparison, but obviously there might be better ways to tweak things.

* I do not believe that Goliath (Ruby), Tornado (Python) or node.js are capable of scaling up to four cores, so to some extent the extra large instance is biased against them. Of course, on the other hand, this is a great demonstration of Haskell's natural ability to scale.

* I may not have chosen the best servers to represent each language. In particular, I have heard mixed reviews of Winstone as a representative Java servlet container.

* The main goal here is to compare web *servers*, not web *frameworks*. However, in the case of Happstack and Snap, the two are tied together very closely. I therefore also included a Yesod test, which tests Yesod + Warp. (You can argue that this isn't a fair comparison for Yesod, since the Snap/Happstack versions to the best of my knowledge are performing no route parsing.)

* And in the near future, Happstack will be switching over to WAI/Warp, so the numbers presented here will no longer be relevant.

So for the moment, I am calling these numbers preliminary. I have given clear instructions on how to reproduce the results on your own EC2 instance: just download setup.sh and run it. I encourage everyone who is interested in this to either improve the results of an existing benchmark, or include new benchmarks in the mix (Erlang and C# have both been requested, I simply did not have time to get to them.)

Also, I would like to include some bigtable benchmarks in the next run as well, but I do not feel skilled enough to write efficients versions of that benchmark in most languages. If people contribute that code as well we can run that next time.
