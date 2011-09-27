Continuing from our [last blog post](http://docs.yesodweb.com/blog/hamlet-lucius-widgets), Sven's benchmark went from being 14 times slower than Rails to 3 times slower. I originally assumed that the database code was the major bottleneck. It turns out that I was *almost* right: there was a much bigger performance leak in the Widget code. But the database code was contributing greatly.

The first thing I wanted to try out was replacing Persistent with direct HDBC code. Sure enough, runtime went from about 12 seconds (for 25 requests) down to 2.5 seconds. As long as I was at it, I decided to test out another theory I've mentioned some times, and switched to properly prepared queries with libpq. To my surprise (and disappointment), it didn't result in any major performance difference. So much for that.

But I was still left with the question: what was slowing down Persistent so much versus raw HDBC? I considered the fact that Persistent was reading every column instead of just the necessary rows. But changing the HDBC code to do the same resulted in almost the same performance. I also tried using quickQuery (and its strict sibling, quickQuery') instead of prepare; also no change.

So I wrote a little test script and went to work on Persistent. If I just prepared the statement, and didn't actually read the data, everything worked out fine. I tried fetching all the rows at once, or using HDBC's lazy functions. __Still__ no change.

I looked at the profiling results again. It said that the [pFromSql](https://github.com/snoyberg/persistent/blob/cf39193fcc9c0de957a5bd179b91693dbd05f8b0/backends/postgresql/Database/Persist/Postgresql.hs#L106) function was being called 220,000 times: 1000 rows * 11 columns * 20 runs. pFromSql converts from HDBC's SqlValue datatype to Persistent's PersistValue datatype. (This level of abstraction is one of the reasons I'd like to switch to a lower-level DB library.)

What caught my eye was the FIXME line of code. If you look at pFromSql, it does most of its work by pattern matching. For the most part, it's a simple translation: SqlString x becomes PersistString x, all types of integers get mapped to PersistInt64, etc. But the last clause uses HDBC's fromSql function to try and deal with any unhandled constructors.

I replaced that line with <code>pFromSql x = error $ show x</code>, ran my test, and got an error message about the <code>SqlLocalTime</code> constructor. I added an extra clause to pFromSql:

    pFromSql (H.SqlLocalTime d) = PersistUTCTime $ localTimeToUTC utc d

and ran my test... it finished almost instantly. I went back to Sven's test case and ran it again: runtime was on a par with the pure-HDBC version, just under 3 seconds (down from our original 12). I don't have a Rails setup on my system, so we'll have to wait for someone else to confirm that Yesod is in fact running faster than Rails on this test.

persistent-postgresql 0.4.0.1 has been released, and I recommend everyone upgrade. And please keep sending in these benchmarks: it's much easier to make Yesod better when we know where it needs improvement.
