For the first time, the Yesod team has decided to make a beta release of Yesod before the official release to Hackage. Between the ease of deployment with Yackage, and the ease of testing with cabal-dev, we feel this will give users a chance to test out the new version without a significant investment of time, and will hopefully lead to some valuable input before the release. It will also give us a chance to focus on updating documentation before the official release.

In order to install the beta copy of Yesod 0.8, please add the following line to your ~/.cabal/config file (or wherever it's kept on your system):

    remote-repo: yesod-yackage:http://yackage.yesodweb.com

You should then be able to just run "cabal update && cabal install yesod", and get the newest version. Feel free to try it out with cabal-dev as well.

I'm announcing a feature freeze as well, excepting two specific issues:

* Changes to Persistent necessary for the MongoDB backend.
* Modifications to Hamlet for $let binding. I'm not sure if this will be included yet, and I don't want it to hold up testing. But since it's a non-breaking change, I have no qualms about adding it later.

I'm going to post the changelog for the relevant packages here. When the official release is made, I will hopefully include a migration guide, similar to what we had for the 0.7 release. But hopefully these notes should be enough to get you started.

## All packages

* The biggest change in this release is a major switch from String to Text. This will probably account for 95% of migration tweaks necessary. We feel that Text is both a better choice performance wise, and a safer fit semantically.

## Lucius

* Nesting. For example, <code>foo { bar { baz: bin; } }</code> becomes <code>foo bar { baz: bin; }</code>.
* luciusFile and luciusFileDebug added.
* @media { ... } support.

## Julius

* Significant performance improvement on parsing large documents. This has no effect on runtime performance, since parsing is performed at compile time.
* As Greg Weber pointed out, there's nothing Javascript-specific about Julius. We've created a new meta-language (in the Text.Romeo module for now) which allows people to create *new* template languages using the same interpolations as provided by Julius. Greg has been testing this out for a very interesting project. &lt;/suspense&gt;

## Hamlet

* Conditional classes, e.g. <code>&lt;a href=@{MyRouteR} :isCurrentRoute MyRouteR:.current&gt;</code>.

## Persistent

* Join modules. For now, supports just one-to-many relations, but we will add more in the future as requested. The modules come in two flavors: application-level join and SQL join.
* select renamed to selectEnum
* Separated out all Template Haskell code into a separate package (persistent-template).
* Under-the-hood changes to make way for MongoDB backend.
* Switch to monad-control from monad-peel. Bas's performance numbers are just too good to ignore :).

## yesod-core

* Upgraded to WAI 0.4
* Attributes allowed on script and link tags created by widget functions.
* Partial file sending.
* addJuliusBody (Javascript inside the body of the page).
* Per-route limits on request body. By default, this is 2 megabytes. See the maximumContentLength method.
* Logging built in. Should be extensible enough to support any backend you throw at it (hslogger seems to be popular for this).
* The Yesod.Core module now exports everything in this package. This is useful when you want to write an app without the entire Yesod framework.
* Add Cassius by media type (addCassiusMedia).

## yesod-json

* Switch to aeson

## yesod

* No longer depends on wai-handler-devel. This means that executables are __much__ smaller. (Note: later releases in the 0.7.* already included this.) You should use the wai-handler-devel executable whenever you want to run in devel mode.
* New mini scaffolded site, which does not include persistent or authentication code. Advantage: __many__ fewer dependencies.
* Scaffolded sites now keep routes and entity definitions in external files. This should allow for add-on tools to automate some boilerplate (i.e., augment your scaffolding).

# For 0.9

I'm hoping to make the official release a little over a week from now. If all goes well, perhaps a week from Sunday (the 17th). My main objectives in that period are documentation and bug fixing.

Speaking of bug fixing: the Yesod test suite is beginning to take shape nicely. I'm planning on having a test added for each bug reported. Anyone who wants to help with the effort of increasing our test coverage is more than welcome. I think this is a very good place for new contributors to start off.

But as usual, the Yesod team likes to keep looking towards the next milestone, in this case, Yesod 0.9. That release will hopefully be our last chance to touch base before the 1.0 release. As such, I'd like to get all of the major changes implemented during this next iteration.

Looking over our [roadmap](http://www.yesodweb.com/blog/road-to-yesod-1-0), we've actually implemented a fair amount of it already. Combining what's left from that list, plus some new ideas that have come up, I think our goals for 0.9 should be:

* Documentation/testing. These will *always* be priorities.
* __New__: an overhaul of wai-handler-devel. The hint package is really wonderful, but I don't think it's up for the challenge anymore. I have more to say on the subject, but I'll leave it for an email to the web-devel list instead.
* A complete i18n solution at the Hamlet/Widget level, tying in to the existing language infrastructure in Yesod.
* Reassessing our forms package. This has been a major TODO list item for a while, and the time has come to finally bite the bullet on it.
* Faster Javascript loading, via something akin to require.js.
* Better static file caching support. This is actually in the wai-app-static package, and can be implemented without breaking changes. If someone out there wants to lend a hand on this, please let me know, I don't think it should be too daunting a task.
