
On behalf of [the Yesod team](http://docs.yesodweb.com/about/), I'm very happy to announce the release of Yesod 0.7. This release incorporates
a large number of community recommended changes, and introduces some new
developers to the team. For those interested in upgrading, please see the [Yesod 0.7 migration guide](http://docs.yesodweb.com/book/migrate6to7/). Some points of interest:

* Yesod now depends on [WAI](http://hackage.haskell.org/package/wai) 0.3, which allows us to use the new [Warp web server](http://docs.yesodweb.com/blog/announcing-warp/). WAI 0.3 uses [enumerator](http://hackage.haskell.org/package/enumerator) and [blaze-builder](http://hackage.haskell.org/package/blaze-builder), which provides for much more efficient response generation than we had previously.

* The yesod package has been split up into a number of sub packages. This makes it easier to try out new ideas without breaking backwards compatibility and more accessible for new developers to help contribute (more on this below). The "yesod" package itself provides a Yesod module that exports the same functionality available previously.

* Static file serving is now more intelligent, providing for directory listings and index files (both configurable), as well as proper trailing-slash support based on folder/file checking. It is based on the newly released [wai-app-static](http://hackage.haskell.org/package/wai-app-static).

* We have new syntaxes for Hamlet, Cassius and Julius. The hamlet6to7 tool is available to help migration.

* yesod-json now uses [json-types](http://hackage.haskell.org/package/json-types), which has a much more robust set of types. In particular, it allows proper creation of booleans, nulls and numbers. yesod-json also provides a compatibility layer for those using the old API.

* The routing system has been rewritten almost entirely. This helps with performance, but there is a much more important consideration here: Code duplication between master and subsites has been removed, making the codebase more streamlined, and we now have the opportunity to make "Yesod middlewares." Expect more on that in the future.

* In addition to Atom feeds, the yesod-newsfeed package also supports RSS feeds. Thanks on this to the new co-maintainer, Patrick Brisbin. You can also have a very RESTful approach that serves either Atom or RSS from the same URL depending upon request headers.

* In general integration with WAI has been improved greatly. There is a new sendWaiResponse function that lets you bypass a lot of Yesod processing, and it is much easier to embed WAI apps directly as subsites. For an example, see the yesod-static package. There will likely be more improvements on this integration in the future, especially since the WAI userbase is growing.

* By default, trailing slashes are not appended to generated URLs.

* newIdent has been moved from Widget to Handler. This is the more correct place for it, as previously it was possible to generate identical identifiers on the same page.

* You can throw exceptions of type ErrorResponse, such as NotFound or InvalidArgs, and Yesod will show the corresponding error page (404 or 400 in this case).

* yesod-auth adds a HashDB backend from Patrick Brisbin.

* We've spun off two non-Yesod-specific packages: [cookie](http://hackage.haskell.org/package/cookie) and [pool](http://hackage.haskell.org/package/pool). Hopefully others will find them useful.

* Provide yesodVersion function for those gracious enough to want to put a "Powered by Yesod 0.7.0" line on their sites.

* Instead of basicHandler, we now have warp, warpDebug and develServer. warp is good for production environments, warpDebug prints some information on each request, and develServer uses wai-handler-devel.

* Installing the yesod package installs all dependencies for scaffolded sites. This was a common point of confusion for new users. The one exception is that neither persistent-sqlite nor persistent-postgresql is installed.

* All scaffolded executables are all built threaded. By default, the production executable is now based on Warp, not FastCGI, though you are still free to use FastCGI in production (I still am for the moment, see LambdaEngine below).

* The scaffold tool checks that entries are valid. For example, it won't let you created a foundation type name beginning with a lowercase letter.

One word of warning: there are two rough areas still with using GHC7:

* The hint package has not yet been ported to GHC7, meaning wai-handler-devel will not work.

* There are still some bugs in the new IO manager which can cause intermittent crashes of applications. I recommend using GHC 6.12 in production until GHC 7.0.2 is released.

## Next steps

As usual, this release is not the end of the road, but just another milestone. To give an idea of what's planned for the future:

* Continue writing the book. My goal is to have the entire basics section (first 10 chapters or so) finished by the middle of March. I appreciate everyone's comments on the book, please keep them coming!

* I consider the "Please Break Yesod" campaign a huge success: we've found a huge number of places where Yesod could be made better for the community. I'm hoping to continue this trend. But assuming that most of the desired changes were addressed in this release, then we can start thinking (again) about a 1.0 release.

* In addition to Cassius, I hope to add Lucius to the hamlet package. Lucius will be a new syntax that is similar to Less. In other words, it will be a superset of CSS3. It will use the same underlying types as Cassius, so it should be easy to swap between the two of them, or use both in the same project. Anyone interested in helping on this is welcome to contact me.

* There will likely be major changes to yesod-form. In particular, I think that polymorphic forms was a bad idea, and I want to investigate basing things on Jasper's [digestive-functors](http://hackage.haskell.org/package/digestive-functors). Because yesod-form is now a separate package, it should be possible to experiment with this new approach without moving to a new version of Yesod. I need to investigate more, but I think it's likely that experimental versions will be released under the package yesod-form-dev.

* I'm looking for co-maintainers on packages. The idea is to spread knowledge around (you know, just in case I get hit by a bus), get fresh eyes to look at problems, and free up some of my time. With the new smaller-packages approach, I think this should be much easier to achieve. If anyone sees a package that they would like to help out with, please send me an email.

* One new project I'm hoping to embark on soon is LambdaEngine. This will be both a library for supporting hot-loading of WAI applications, and a hosting service for WAI apps. My initial goal is to migrate my current set of applications to a new server using this (you may have noticed that my web pages get downtime occassionally, it's almost always that my node has "had issues"). I'm hoping to make this as easy-to-use as possible, and possibly even provide free hosting for apps, though we'll need to see how that works for me budget-wise. Don't expect any big announcements on this immediately, but if you're interested in the idea let me know.
