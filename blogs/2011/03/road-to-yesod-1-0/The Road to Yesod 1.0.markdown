Our personal and professional lives sometimes make demands on us that slow down our dreams.
The frantic pace of Yesod development temporarily slowed to mostly just keeping up with patches.
But the slowdown made everyone reflect on where we are going to focus our effort now that we pick the pace back up. 

## Seven of Nine

<img src="/static/7of9.jpg" alt="Seven of Nine">

We are very proud of the 0.7 release.

* The new hamlet syntax appears to be a hit. Once you write html this way you won't want to go back to normal templating.

* The modularity introduced makes future developments much easier.

* Coinciding With the [release of the *fast* Warp web server](http://docs.yesodweb.com/blog/warp-speed-ahead), deployment got much nicer, and [WAI](http://docs.yesodweb.com/book/wai) got a lot more attractive. By the way, GHC 7.02 was released with important bug fixes- please deploying Warp with GHC 7.02.

We will continue making 0.7 point releases, and are planning a 0.8 and 0.9 releases before making a 1.0 release.

# 1.0 changes

## Documentation

Perhaps the biggest piece of feedback we have received is a need for better documentation.
We have been making steady progress on this, but a 1.0 means a completion of the [Yesod book](http://docs.yesodweb.com/book).


## JSON

In our last post we mentioned plans to develop json-enumerator.
In the meantime, Bryan O’Sullivan sped up his [just released json package called aeson](http://www.serpentine.com/blog/2011/02/25/faster-better-cleaner-new-aeson-and-attoparsec-releases/).
With Bryan's name behind this fast, feature-rich package, we feel it is going to become the default JSON package.
As much as we love enumerators, we feel that integrating with this package is the right move.
If the need arises we should be able to place an enumerator interface on it in the future.

## Testing

It is easy to get lulled into a false sense of security by Haskell's types.
But the reality is that when you are developing something as complex as a web framework, there are going to be bugs.
We know that we can save time by writing tests. They find bugs, and they document code.
One of the reasons for a lack of testing is a lack of convenience.
Testing code should be as easy as writing it.
Now that cabal can build test suites for a package, a big obstacle has been removed. You have to compile from head for this feature, but it should be released soon. Hopefully cabal-dev will support this soon.
The recent release of wai-test makes testing WAI application much easier.
It is already being put to use and discovering bugs.

Testing isn't just for a framework, but also for framework users.
We want to provide a testing setup in the scaffolded site, along with testing helpers to lower the barrier to testing for Yesod users.


## Templating

### Cassius, meet Lucius

Currently we provide a whitespace-sensitive syntax for creating CSS. While this works well for many cases, it has hurt us in two ways:

* You can't simply copy-paste your CSS content into a Cassius file.

* It is very tricky to properly support some more advanced CSS-template language features, such as nesting.

Instead of ditching the whitespace syntax of Cassius, we've decided to augment it with a brace-style syntax called Lucius. We'll be looking for community input on which syntax is nicer, but for the moment the plan is to support both.

At the same time, we plan to add first-class support for media types, and include more CSS helper functions. We already provide data types for colors and unit measurements, and intend to add more as time goes on. This is also a great place for new contributers to get started.

### Generalized Julius.

Julius (javascript templater) is a pass-through filter- it knows nothing of the semantics of the file, other than that it needs to make substitutions.
This can be generalized to template any file. This is similar to how StringTemplate is used now, but you can enjoy type safety and much more convenient interfaces.


## Forms

We are still strongly considering a major overhaul of the forms package. In all likelihood, we'll be migrating it to be based on the digestive-functors package. We are hoping to avoid any major API changes in the process, but hope to gain:

* more human-readable compile-time error messages

* a stronger base, with more opportunities to interoperate with other packages.

## Faster javascript

Loading javascript files from the head tag blocks page loading. Particularly in a high performance framework like Yesod, this can become *the* bottleneck. We are going to be adding asynchronous javascript loading, like [require.js](http://requirejs.org/). As with other things Yesod, we plan on supporting any client-side library a programmer wishes to use, but pick the best option available for the recommended approach.

## Caching support

What good is a fast server if files aren't being cached?
Instead of needing to setup caching on a proxy server, wai-app-static will add caching headers itself.
At a minimum this is useful for creating a more production-like environment when doing local development.
But even more significantly, this clears the way for a *pure* Warp deployment with no proxy server.
This lowers the barrier to entry, but may not be for everyone- Apache and Nginx will always offer a lot more features. As always, by being built on WAI, Yesod lets you keep your deployment options open.

## Persistent

The MongoDB backend compiles, but hasn't been given a final push just because our priorities have been elsewhere. It will be ready for the 1.0 release. There is also work going on for a MySQL backend, and discussions of sharding and caching support.

By the way, if anyone is interested in helping out, the PostgreSQL backend would like to switch from HDBC to using the C API directly. We think this will offer some major performance advantages. If you'd like to help out, let me know, this is a great way to work on a small corner of Yesod with huge benefits for all its users.

## Internationalization

There have been a few discussions and blog posts regarding i18n support. We have purposely avoided putting in a complete i18n solution so far, since we did not think the design space was properly explored yet. At this point, I think we have enough information to make an informed decision. Type-safe, powerful I18N will be a cornerstone feature of Yesod 1.0.

## Scaffolding

The current scaffolding tool generates a very powerful project with Persistent and authentication support. While this is very nice, it can be overkill for a lot of projects. We're planning on adding an extra scaffolder that will produce a simplified project with less dependencies.

## Better support for static html

We said above that Hamlet is addictive: its light-weight syntax and type-safe URLs are hard to beat. So much so that people want to use Hamlet for creating their static pages as well. We're looking into better support for this.

## Backend changes

What's the right datatype for a URL? There's the URI datatype defined in the network package, but what about when you want to send it over the wire, like in a redirect response? String and Text are not correct, since you can't use arbitrary Unicode charcters. ByteString isn't correct, because it's textual data (and cannot have the eighth bit set).

Problems like this have prompted the creation of the ascii package. We're planning on evaluating a number of uses of ByteString and String throughout the Yesod ecosystem, such as yesod-core and mime-mail, and replace them with the Ascii datatype. This is more semantically correct, and will hopefully help avoid bugs.

This change will also affect http-enumerator... oh, and while we're on the subject, http-enumerator is getting keep-alive support soon. That's going to make the best Haskell HTTP client solution even better.

# Implementing The Road-map

This list does not exclude anything. Some things are more tentative than others, and it will keep evolving as we listen to feedback. We just want to communicate to Yesod users where we are planning on focusing our efforts and what they can expect in the future.

The Yesod community is growing, and the number of casual contributors is growing.
If you have a need and are willing to write a patch for it, there is no reason why it can't be in the 1.0 release. We will work to help you understand the code, review your patch, and get it included.
Even if you don't have a burning issue, you are welcome to come hack with us on a real-world code base that is large, but still fun Haskell code. It is a great way to learn about haskell or web programming.

We are excited to see a 1.0 on the horizon. But 1.0 is just a lonely number taken to the first decimal place- we attach two meanings to 1.0. The first is API stability. The second is a *complete* framework- you won't come across any major weaknesses. As with any Yesod release, you can can expect a safe, fast, and highly productive framework.
