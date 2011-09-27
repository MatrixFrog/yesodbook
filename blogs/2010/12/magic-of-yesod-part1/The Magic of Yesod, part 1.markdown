Thanks to everyone who responded to the [Yesod user survey](http://docs.yesodweb.com/blog/yesod-user-survey/). The feedback I have gotten has helped me restructure my [Yesod TODO list](http://wiki.yesodweb.com/TODO%20List) and given my ideas on how to restructure the docs site. Though I will not be releasing all of the results of the survey (I always ask for participants permission beforehand, especially if I ask for personal details), there are some results from the survey that I will be releasing in the future.

For the moment, I will share with everyone that the number one request was improved documentation. (By the way: I agree on this, it just takes a long time to write quality documentation.) Most people seemed to like the book best of all documentation sources. One area in particular that people wanted clarifications about was the "magic" of Yesod. This basically breaks down into the following:

* Hamlet/Cassius/Julius templates
* The routing quasi-quoting (parseRoutes)
* mkYesod (goes together with parseRoutes)
* Persistent entity declarations
* Persistent automatic migrations

I *think* that covers all the major uses of Quasi-Quotation and Template Haskell in Yesod. If I've left any out, please let me know. The point of this post is to explain *how* this stuff works, and to be the basis upon which to build a chapter devoted to the topic. Even though this goes in the "advanced" category, and I'd rather finish the basics part of the book first, I think this is an important enough topic to warrant going out-of-order.

## But what *is* this magic?

For the uninitiated, let's first cover what these two extensions are. Template Haskell allows you to write Haskell code using Haskell code. You have datatypes representing expressions, literals, types... everything. You can build these up in the Q monad to create anything you want, including defining new datatypes, new functions, and type instances.

Now that Q monad I mentioned has two very important properties for our purposes: you can run IO actions inside of it, and when you wrap a Q value inside a Template Haskell splice, it gets run by the compiler to generate the appropriate values. As a simple example:

    {-# LANGUAGE TemplateHaskell #-}
    import Language.Haskell.TH

    main = putStrLn $(litE $ stringL "Hello World!")

Wrapping up the Q value inside $( and ) is how you "execute" the TH code. Oh, and since you can do IO actions inside the Q monad, you can have your TH depend on the contents of an external text file. That can be really cool, but one word of warning: GHC will **not** know to recompile your code if that external file changed.

As for quasi-quotation, it uses the same Q monad and Template Haskell datatypes, but allows you to embed some text inside special brackets. This essentially lets you embed an arbitrary syntax inside Haskell code, which can be very cool. A simple example of usage would be:

    -- Multiline
    module Multiline where

    import Language.Haskell.TH
    import Language.Haskell.TH.Quote

    multiline = QuasiQuoter
        { quoteExp = litE . stringL
        }

    -- some other file
    {-# LANGUAGE QuasiQuotes #-}
    import Multiline

    main = putStrLn [$multiline|
    This is a multiline
    string.
    |]

Two important points:

1. You need to define the multiline QuasiQuoter in a separate file due to GHC stage restrictions.

2. In GHC 7, the syntax for quasi-quotation has changed, so our main function would become:

    main = putStrLn [multiline|
    This is a multiline
    string.
    |]

Hopefully that should be enough of an introduction to TH to understand the rest of the discussion. But as with any kind of metaprogramming, TH can be very complicated to write, and usually even more difficult to read. The Haddocks will be your friend here.

## Hamlet/Cassius/Julius templates

These are the least magical of all. There are three versions of functions for each language: a quasi-quoter, a TH function and a TH debug function. Let's start with the simplest of these: the Julius quasi-quoter. It does the following steps:

1) Given the String input, parse it into raw text, escaped operators (%%, @@ and ^^) and into interpolations (%myVar%, @myUrl@, \^myJulius\^).

2) For the interpolations, convert them into references to Haskell values. For example, the interpolation %(foo.bar).baz% would be converted into <code>(foo bar) baz</code>.

3) A Julius value is a function taking a **URL rendering function** and returning a Javascript value. Javascript is just a newtype wrapper around a blaze-builder Builder value, for efficient building up of ByteStrings. We now have four types of values (raw strings, %variable interplolations%, @url interpolations@ and ^template interpolations^). These are all converted to a Julius, by:

    * Raw strings: UTF8-encoding the data and converting to a Builder. This Builder is wrapped in a Javascript, and const is applied to turn this into a Julius.
    * Variables: The variable must be an instance of ToJavascript, therefore toJavascript is applied, followed by const.
    * URLs: The URL rendering function is applied to get a String, followed by conversion to Builder and then Javascript.
    * Templates: The value must already be a Julius: no action taken.

4) Since a Julius is an instance of Monoid, we simply mconcat all of these pieces to get our final answer.

The Template Haskell version does almost the exact same thing, except it reads the contents of the template from a file. Remember, you can run IO actions inside the Q monad. Unfortunately, this means you will need to recompile your code every time you change your Javascript, which can be annoying during development. Therefore, the debug version reloads the contents of template on each call. This involves some fancy footwork to make sure the correct variables are available, but nothing too scary.

Cassius is almost exactly the same thing, except:

1) The content is parsed for the whitespace-sensitive layout Cassius uses, and then rendered to proper CSS afterwords.
2) There is no template interpolation in Cassius.

And Hamlet is just a step above that, introducing things like foralls and conditionals. However, all of this is just following the concept of building up Haskell functions using TH code, and referencing the variables in scope when the template was called.

*If anyone has specific questions about how all this works, please ask in the comments and I'll try to answer.*

## To be continued

Unfortunately, it's late over here, so I need to cut this post short. Stay tuned for later posts as I'll try and explain the rest of the magical Yesod pieces.
