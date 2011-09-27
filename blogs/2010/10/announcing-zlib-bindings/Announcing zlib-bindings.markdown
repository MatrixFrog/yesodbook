I just released version 0.0.0 of the [zlib-bindings package](http://hackage.haskell.org/package/zlib-bindings), which I think fills a nice gap in the compression realm for Haskell. Up until now, we've had the [zlib package](http://hackage.haskell.org/package/zlib), which provides a nice, high-level interface to compression based on lazy bytestrings. Unfortunately, there are two problems:

1) This depends on lazy IO, which many people try to avoid.

2) There is no way to use this library when working with streams.

The second point forced me to write some FFI code into the [wai-extra package](http://hackage.haskell.org/package/wai-extra) to implement the Gzip middleware. More recently, I decided to add compressed HTTP support to [http-enumerator](http://hackage.haskell.org/package/http-enumerator), and was facing the same challenge. Instead, I decided to simply pull all of that code into a single package.

The package uses the zlib terminology of deflate for compression and inflate for decompression. There are API docs, but I think the easiest way to get started is to simply read the [unit tests](http://github.com/snoyberg/zlib-bindings/blob/master/test.hs). Here are two samples: the first performs deflation/compression, the second inflation/decompression:

    deflateTest = do
        license <- S.readFile "LICENSE"
        def <- initDeflate 7 $ WindowBits 31
        gziped <- withDeflateInput def license $ go id
        gziped' <- finishDeflate def $ go gziped
        let raw' = L.fromChunks [license]
        assertEqual raw' $ Gzip.decompress $ L.fromChunks $ gziped' []
      where
        go front x = do
            y <- x
            case y of
                Nothing -> return front
                Just z -> go (front . (:) z) x

    inflateTest = do
        license <- S.readFile "LICENSE"
        gziped <- S.readFile "LICENSE.gz"
        inf <- initInflate $ WindowBits 31
        ungziped <- withInflateInput inf gziped $ go id
        final <- finishInflate inf
        assertEqual license $ S.concat $ ungziped [final]
      where
        go front x = do
            y <- x
            case y of
                Nothing -> return front
                Just z -> go (front . (:) z) x

The package is tested on both Linux and Windows, and since it uses the zlib package internally, it does not introduce a system library dependency on Windows. Hopefully this package will be of use to some people.

I also simulataneously released a new version of wai-extra which uses this library internally, and http-enumerator 0.2.0 which includes gzip compression support. This new version of http-enumerator includes some minor fixes, and also consolidates all exceptions into a single type (HttpException).

Following the dominos along, I've also released authenticate 0.7.0; the only changes there are a bump to http-enumerator 0.2.* and also combining all exceptions into AuthenticateException. This is the library underlying the login mechanism for [haskellers.com](http://www.haskellers.com/), and so has seen a good amount of testing recently.

And finally there's new releases of [yesod](http://hackage.haskell.org/package/yesod) (0.5.2) and [yesod-auth](http://hackage.haskell.org/package/yesod-auth) (0.1.2) which accounts for the new authenticate version. There are also a few minor additions to yesod; I won't bore you with details here, you can check out the log on github if you're interested.

## Yesod 0.6

I'm still confident that we are fast approaching a 1.0 release of Yesod. My guess is that 0.6 will be the last release beforehand. The goal for that release is to separate off some tangential aspects into separate packages. I've already mentioned yesod-auth; I'll probably also release Yesod.Mail as a separate yesod-mail package. I'm also considering adding support for markdown emails and a few other features that could be nifty.

I've said it before, but I'll throw it out there again: if you have any feature requests for yesod, it'll be easier to add them in now versus later. You can post your ideas in the comments here, on the web-devel list or just email me directly.
