By Greg Weber

# Yesod 0.8

For a change-log see the [beta announcement](http://www.yesodweb.com/blog/beta-yesod-0-8), although we sneaked in some features since the beta release. The [migration guide](http://www.yesodweb.com/book/migrate7to8) goes over what you need to change in your application.

Originally Yesod 0.8 was mostly about internal consistency- the main goal was to remove the use of String and instead using Text or ByteString everywhere. But some nice features have managed to get into this release.

With every release of Yesod, we see more new contributors. Thanks to Chris Casinghino: he caught 3 bugs in the beta release (1 of them being a GHC bug). The other two required breaking API changes to fix. Also a big thanks to Rick Richardson- without his help we wouldn't be able to announce perhaps the biggest feature of this release: preliminary support for MongoDB in Persistent! This was a great example of collaboration in Yesod. There was an original implementation written quite some time ago. Rick finally picked it up and pushed it to completion, consulting with us along the way.

Also, thanks to Aur Saraf for the initial SQL join code, and for contributing a lot of good ideas for the final design.

## Persistent

### MongoDB support
Persistent is a universal, type-safe data store interface for haskell. Persistent was always designed with newer databases (NoSQL) in mind, but only a sqlite and postgresql backend were implemented. The addition of the MongoDB backend forced some under the hood changes, but has validated the extensibility of persistent. And it is a great case of the advantages of Persistent. By default, MongoDB is schema-less, and the haskell driver supports this as it should because some use cases can take advantage of this. However, for most use cases there is a known schema, and you are always a typo away from inserting the wrong key or querying a key that doesn't exist. With Persistent you know at compile time this won't occur. You also get automatic conversion from the driver type to a normal Haskell ADT that you define.

This is an alpha release. Some important features of MongoDB do not have full support yet. In particular, while we support embedded lists, maps, and primitives, embedding another ADT is not yet supported.

If you haven't tried MongoDB yet, it is perhaps the most feature-rich NoSQL database, and a great general purpose web application database for applications that don't need transactions. The main motivation for its use instead of a traditional SQL database are speed and easier scalability. The 1.8 release of MongoDB adds single-server durability, so it is still a great choice for small-scale web applications. 

### Join modules
Persistent is designed for high scalability- we have always thought about application-level joins first. This release adds both application level joins and SQL joins- take your pick and change it up when the need arises.

### Future work
As mentioned above, there is still work to be done to fullly support MongoDB. More types of joins can be added also.
Users generally want more database specific features. While we can add these features, it is also important for Persistent to make it easier to drop down and run queries that it cannot generate, but still receive haskell ADTs in response, or to support the addition of raw queries to what it generates. Persistent is a great library for the haskell community, and adding the next data store is always a fun, self-contained project to embark on. One direction we have always been thinking about is having key-value stores support the key-value portion of the Persistent API. I want to stress that while most of Persistent development does occur within the Yesod community, there is nothing specific to Yesod about it. And there is nothing stopping you from using it in a different web framework, or on a project that has nothing to do with web development.

## Template languages

### Miscellaneous
* Attributes are now allowed on script and link tags created by widget functions.
* addJuliusBody (Javascript inside the body of the page).

### Lucius is now a first-class css templating language

The Hamlet template family originally include Cassius- a way to tersely describe css using whitespace layout.

    table tbody
        width: 100px
    table tr
        height: 20px

The advantage of Cassius is a simple syntax that bypasses the need for brackets and semi-colons. We do love cassius for writing new css, but there is a cost to re-formating old css or re-formating cassius to normal css for use elsewhere.

Lucius uses familiar hamlet-style variable insertion, but it is a superset of css. The goal is that you can copy and paste any valid css into a lucius template. This release improves css compatibility and gives lucius the benefit of nesting:

    table {
      tbody { width: 100px; }
      tr { height: 20px; }
    }

### Hamlet adds conditional classes
Hamlet is really a joy to work with, but we still find things to improve. This release adds conditional classes. The following will conditionally add a class of "current".

    <a href=@{MyRouteR} :isCurrentRoute MyRouteR:.current>.

### Generalizing the Hamlet family of templates
At its core, the Hamlet family of templates are a new (for haskell at least) style of compile time, type-safe, easy to use templates.
Hamlet templates automatically capture the variables in their environment so you don't have to waste time re-defining what is inserted. This makes their ease of use on par with dynamic languages, but we can do all the parsing at compile time and still have type-safe insertions.

Hamlet (html), cassius, and lucius (css) have specific parsing rules. However, Julius (javascript) has always been a pass-through template: variables are interpolated, but the source text is unmodified. We realized this was a starting point to extend the Hamlet family to other file types, and re-factored Julius. There is now a coffeescript (great language that compiles cleanly to javascript) module, and adding a template over any new file type is a fairly straight-forward task. The variable parsing is configurable. In javascript, it is `#{var}`, just as in hamlet or lucius, but in coffescript it is `%{var}` to avoid conflicts with coffescript syntax. We plan on using this to add a simple html template for those who either need to maintain better compatibility with existing html or are not yet enlightened about the joys of using hamlet.

Currently the templates are geared towards being used in Yesod- they support type-safe url insertion. When we get around to it we will release a version without url insertion so that users outside of Yesod have no extra overhead. Hamlet-style templates are by far the best haskell has to offer, with one drawback: given that the templates are parsed at compile-time, it isn't always convenient to reload them when developing. In Yesod, we are solving this (and re-loading in general) by creating a smarter development environment.


## Development environment reloading
A new version of wai-handler-devel is underway that will efficiently reload code in development mode. This will finally get us out of the "haskell straight-jacket" and allow an ease of development on par with dynamic languages. Coinciding with this change is a change in the yesod executable.

### the yesod command
* Running "yesod" by itself gives you a list of commands.
* Running "yesod init" gives the behavior previously held by "yesod", i.e. generate a scaffolded site.
* Running "yesod build" is *almost* identical to "cabal build", but with one change: it performs a dependency analysis of external files included by Template Haskell (Hamlet templates, routes, entity definitions) and changes modification times as needed to force cabal to build modules. For example, if "Handler/Root.hs" references "hamlet/root.hamlet", and the latter has a later modification time than the former, the former's modification times will be changed to match that of the latter.
* Running "yesod devel" runs the devel server (automatically re-compiles code) which now uses cabal for the compiling (passing in a special "devel" flag). Previously we were using the hint package (uses the ghc api) and re-loading the entire application. The new version uses direct-plugins (uses .hi files) and will load individual files.

If this sounds like overkill to you, keep in mind that best practice in dynamic languages is to have a file watcher that will run the test suite as changes are made (based on user defined rules that need to be adjusted) and that development mode (re-interpreting files when a request comes in) can be noticeably slow. By using (the type-safe language) haskell we have avoided the need to constantly run a test suite. But we need some smarts to achieve fast re-compilation. This project watching tool is another that will hopefully be shared at least among other web frameworks, if not in haskell projects in general.

## JSON support
We are switching our default JSON library to the excellent aeson package. This adds speed, features, and easier compatibility if this package becomes the community's default json package. We are giving up enumerator streaming, but that should be addable to aeson when the need arises.

## Miscellaneous features
* Partial file sending.
* Per-route limits on request body. By default, this is 2 megabytes. See the maximumContentLength method.
* The Yesod.Core module now exports everything in its package. This is useful when you want to write an app without the entire Yesod framework.
* Logging support built in. This should be extensible enough to support any backend you throw at it (hslogger seems to be popular for this). There is still work for us to do at least to document how to log efficiently in a production envrionment. The key will probably be to log to stdout/stderr and re-use good existing logging tools.
* No longer depends on wai-handler-devel. This means that executables are much smaller. (Note: later releases in the 0.7.* already included this.)
* New mini scaffolded site, which does not include persistent or authentication code (many fewer dependencies).
* Scaffolded sites now keep routes and model definitions in external files. This should allow for add-on tools to automatically generate code for you.

## The future: 0.9 - 1.0
* Documentation- we know that this is really the biggest area for improvement in Yesod. By the 0.9 release we will probably make some improvements to the documentation site to make it easier for others to contribute. And we will hold up the 1.0 release until we have documentation that we can be proud of.
* static file caching headers support - a rough implementation is already completed. This allows for a pure haskell deployment- Apache or Nginx are optional.
* Support for other template types as first class citizens. So you can treat coffeescript as if it were javascript or use a different html template in the same way you would use hamlet.
* Easier support for faster, non-blocking Javascript loading via something akin to require.js.
* A complete i18n solution.
* Reassessing our forms package, probably to use digestive functors. Yesod forms currently use polymorphism- that is a great feature, but the limit of this abstraction, like many other compile-time features in haskell is often the ability to decipher error messages. That is easy enough to deal with when you own the code, but when someone else is interfacing with your libraries, error messages are a part of that interface.
* Embedded objects in MongoDB. We are leaving other persistent features off the official roadmap, but expect other improvements.
* Performance improvements. We don't have anything specific planned here, but we eagerly tackle any solid reports of slow spots in the framework. We will probably take a closer look at performance aspects post 1.0

0.9 is a chance to finish all the features that we feel are needed for a 1.0 release, and then to have another cycle to reflect and make sure things are done right, well-documented, and that we aren't missing anything that we view as critical for a complete web framework.

As always, if you send a patch for or start working on a useful feature, we will make helping you our top priority, and we are always listening to the needs of Yesod users- so there is always a lot of variance in the roadmap.
