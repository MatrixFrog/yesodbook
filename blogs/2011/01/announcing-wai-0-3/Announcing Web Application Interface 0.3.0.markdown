The Web Application Interface (WAI) is a low-level, highly performant, extrmely generic layer between web applications and web servers. It allows you to write your application once and run it on a development server, production server, FastCGI and even as a desktop application.

This new release introduces a large change in philosophy: previous versions only depended on the base and bytestring packages. Now, for both performance and ease-of-use, we have introduced three new dependencies:

* enumerator: Anyone who follows this blog probably knows I'm a fan of John Millikin's enumerator package. Previously, WAI used a built-in Enumerator datatype for the response body, and a custom Source datatype for the request body. Now everything is provided in terms of enumerator's datatypes, making coding against the WAI much easier and providing a wealth of tool support.

* blaze-builder: I'm also a fan of Simon Meier and Jasper Van der Jeugt's blaze-builder package. This essentially lets you construct optimally sized ByteStrings with a minimum of buffer copying. Switching to this can provide the WAI with huge performance advantages.

* network: You might be surprised that this wasn't a dependency in the first place. It is included now so that remoteHost is represented by a SockAddr instead of a ByteString, which means less DNS lookups in general. Thanks to Aur Saraf for the recommendation.

There is also a large change in how a Response works. In particular, you are now able to generate the status and response headers from within the response enumerator, which allows us to create things such as proxying web sites. A proper example of this is below. If people are interested in more details of the API changes, let me know in the comments and I'll write up a post about it.

As usual, a new WAI release comes with new releases of the wai-handler-fastcgi and wai-handler-scgi packages, as well as a new release of wai-extra. There are a few things to note in this release of wai-extra:

* A new autohead middleware, which automatically produces responses to HEAD requests based on your response to a GET request.

* A new vhost middleware, which allows you to route requests to different applications based on hostname (or anything else you want for the matter).

* SimpleServer is gone.

## Welcoming Warp

A few people over the past years have contacted me and asked whether SimpleServer could be used in a production environment. One of these people, Matt Brown (one the core Yesod team) actually *has* been using it in production for a while, and decided to finally push it from a test server to a full-fledged production server.

The result of this is Warp. We will likely be releasing the server in the next few days. I won't go into too many details now, but some nice features of it are:

* it's programmed in a fairly high-level manner, relying mostly on enumerator and blaze-builder for the heavy lifting

* it's very fast

It turns out servng with Warp is significantly faster than any of the current alternatives, so after Yesod 0.7 is released Warp will become its recommended deployment option.

## wai-test (and it's about time)

Greg Weber (the third Yesod core developer) has been pushing for more testing for a while. The result is wai-test: a very simplistic HTTP unit test framework for WAI applications. It currently does not have very many features, but these will likely be added as we increase our Yesod test suite. wai-extra already uses wai-test for checking all of its middlewares.

## http-enumerator

I wrote a blog post about month ago about the [serendipity between wai and http-enumerator](http://docs.yesodweb.com/blog/serendipity-wai-http-enumerator/). With http-enumerator 0.3, we're now using the same datatypes (Status and CIByteString) as WAI. This makes http-enumerator more correct (response headers should be case insensitive) and more informative (you now have the status message in addition to the status code). Plus it lets you write the fabled web site proxy (uses the not-yet-released Warp):

    {-# LANGUAGE OverloadedStrings #-}
    import qualified Network.HTTP.Enumerator as H
    import qualified Network.Wai as W
    import Network.Wai.Handler.Warp (run)
    import qualified Data.ByteString.Char8 as S8
    import qualified Data.ByteString.Lazy.Char8 ()
    import Data.Enumerator (($$), joinI, run_)
    import Blaze.ByteString.Builder (fromByteString)
    import qualified Data.Enumerator as E

    main :: IO ()
    main = run 3000 app

    app :: W.Application
    app W.Request { W.pathInfo = path } =
        case H.parseUrl $ "http://wiki.yesodweb.com" ++ S8.unpack path of
            Nothing -> return notFound
            Just hreq -> return $ W.ResponseEnumerator $ run_ . H.http hreq . go
      where
        go f s h = joinI $ E.map fromByteString $$ f s $ filter safe h
        safe (x, _) = not $ x `elem` ["Content-Encoding", "Transfer-Encoding"]

    notFound :: W.Response
    notFound = W.responseLBS W.status404 [("Content-Type", "text/plain")] "Not found"

Note that you do *not* need to actually be using WAI to use the new http-enumerator; the dependency is simply there to provide datatypes. And since http-enumerator already depends on all of WAI's dependencies, you only need to install one extra package.

There are many more changes for http-enumerator in the pipeline. Gregory Collins pointed out that persistent connections could be very useful, and Iustin Pop has requested support for controlling SSL certificate checks. If anyone would like to volunteer to help implement these features, I'd appreciate it: I'm a bit swamped at the moment.

## authenticate 0.8

This new version of authenticate uses the new http-enumerator, and fixes some API problems in the OpenID code. Jeremy Shaw pointed out that there was no support specifying extra parameters or receiving extra parameters. Additionally, error message handling was a bit weak and some sites require the realm to be set. Thanks to Jeremy, authenticate should now work with even more OpenID providers.

## Coming up

This was the first batch of package releases. I have a number of batches on the way. As I said above, I'll be releasing the Warp web server soon. This will possibly be accompanied by a not-yet-written package for a generic static file server and a warp-based static file server. Along with warp I can release a new version of wai-handler-devel.

After that, I want to finally tackle Hamlet 0.7. There have been a lot of proposals put out, and it's time to make some decisions and implement things. Persistent probably won't see any major changes in the 0.4 release besides some dependency bumps and removing some deprecated functions.

And then comes the Yesod 0.7 release. As I've mentioned before, this release will see the main package get split up into smaller packages. So far: yesod-core, -form, -json, -newsfeed, -persistent, -sitemap and -static. We'll also have a "yesod" package to provide the scaffolding tool and tie everything together. And when all that's done, I can finally get back to finishing the basics part of the Yesod book.

Moral of the story: if you have any ideas you want implemented in any of these packages, let me know soon.
