I'm very happy to announce a new release of the [Yesod Web Framework](http://hackage.haskell.org/package/yesod). This release is accompanied by version bumps in [Persistent](http://hackage.haskell.org/package/persistent) and [yesod-auth](http://hackage.haskell.org/package/yesod-auth), as well as a massive reorganization of the content in the [Yesod book](/book/), hopefully making the material easier to follow and more comprehensive. The main goal in this release has been pushing forward the stability of the Yesod API, paving the way for a 1.0 release in the near future. Some notable changes in this release are:

* Changes to the session are visible immediately, instead of needing to wait until the next request. This has been a source of confusion for a few people, and it turns out the new implementation is simpler. I just [wrote an email about it.](http://www.haskell.org/pipermail/web-devel/2010/000508.html)

* CSRF protection for forms. A bit more information on that in [another email I wrote](http://www.haskell.org/pipermail/web-devel/2010/000507.html).

* A major renaming of the widget functions for simplification. You can [read the email I wrote about it](http://www.haskell.org/pipermail/web-devel/2010/000501.html).

* Completely replaced MonadCatchIO with MonadInvertIO. The former had a flaw in its implementation of finally which allowed database connections to be leaked. The latter has a full test suite associated with it, and should be safe.

* The Auth helper has been moved into the yesod-auth package (version 0.2 just released). This slims down the dependencies in Yesod while allowing us to have more freedom in changing the API. yesod-auth deviates from the previous Auth helper by providing a plugin interface, allowing anyone to add new backends.

* Moved the Yesod.Mail module to the new mime-mail package, allowing its use outside of Yesod.

* Added support for free-forms. In addition to the formlet-inspired, Applicative form API, we now have a second datatype (GFormMonad) which makes it easier to write custom-styled forms. I [wrote a blog post about this](http://docs.yesodweb.com/blog/custom-forms/) already.

* The jQuery date picker can now be configured through a Haskell datatype.

* The Widget type alias has been removed; now the site template defines this.

* mkToForm can be applied to a single datatype instead of all of your entities.

* Added fiRequired to FieldInfo, making it easier to style your forms differently for required and optional fields. We also have a fieldsToDivs function for those who hate tables.

* By default, sessions are tied to a specific IP address to prevent session hijacking. Under certain circumstances, such as proxied requests, this can cause a problem. There is now a sessionIpAddress method in the Yesod typeclass to disable this.

I'm very encouraged by the fact that most of these changes were incremental enhancements on features already present, and that even those often did not require breaking changes. My goal for the near future is to continue focusing on the [Yesod book](/book/). I'm hoping to have the majority of the book written by the time we have a 1.0 release.

The code should be pretty stable: it's already running [haskellers.com](http://www.haskellers.com/), so it's getting a fair amount of stress testing. If you find any bugs, let me know. And if you have any questions, ask away!
