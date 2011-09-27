
We're happy to announce the 0.4.0 release of [WAI](http://hackage.haskell.org/package/wai), along with many of its
associated packages, including [Warp](http://hackage.haskell.org/package/warp). WAI is the Web Application Interface,
providing a uniform layer between applications and backends. WAI is the
foundation of Yesod and the future foundation of Happstack, and also
provides backends for non-framework web applications. Warp is the premiere WAI backend, providing a
[very fast](http://www.yesodweb.com/blog/preliminary-warp-cross-language-benchmarks) standalone server.

While the overall interface is almost identical, this new release provides a
few nice enhancements:

* Instead of defining our own datatypes, WAI uses the [http-types](http://hackage.haskell.org/package/http-types) and [case-insensitive](http://hackage.haskell.org/package/case-insensitive) packages. (Thanks to Aristid and Bas.) In addition to types, http-types provides a number of convenient functions, such as query parsing/rendering. Additionally, many of the special values, like all the status values, are now provided by http-types instead of WAI, making the Network.Wai module much smaller.

* The queryString and pathInfo records in Request have been renamed to rawQueryString and rawPathInfo, and parsed versions of these fields are now provided under the previous name. This should allow middlewares to function more efficiently (they won't all need to parse the same query string), and assist with deferred routing.

* The errorHandler record was removed. It was poorly named, not implemented well, and never used. Error logging should be handled at the application level instead.

* Support for partial file support was added.

For the most part, associated packages have had little to no changes in their
external behavior, outside of adapting to the new version of the interface. For
Warp, I'd just like to point out two changes:

* There is now a settingsHost setting, which allows you to __specify which host to bind to__. Previous versions of Warp simply used the listenTo function. The default behavior is to listen on all hosts.

* The previous set of run* functions were a bit of a hodge-podge. We now have consistent naming: run, runSettings, and runSettingsSocket.

## Yesod

For those curious about Yesod: the 0.8 release will support this new version of
WAI, and will hopefully be released within the next week and a half. The
upcoming release is mostly about smoothing out some type decisions in APIs, and
adding some requested features throughout the codebase. The new version should
hopefully introduce very few breaking changes.
