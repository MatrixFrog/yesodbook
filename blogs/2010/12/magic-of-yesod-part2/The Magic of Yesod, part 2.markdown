For those celebrating one, a happy holiday. To everyone else, a happy day.

Before we get started on the second installment in this series, I wanted to give everyone a little update on Yesod 0.7. Felipe Lessa came up with a very good idea: split up Yesod into a yesod-core package which will provide basic functionality and have minimal dependencies, and a yesod package which will tie together a bunch of addon packages. This will help out in a few ways:

* The scaffolded site will no longer have extra dependencies versus the yesod package.

* The scaffolded site can be a little smaller, since some of the functionality that was included there can be put in the yesod package.

* One of the goals of the 1.0 release is to have a complete test suite. Breaking things up into smaller packages makes it (at least psychologically) easier to test the whole thing.

* Make it easier to make breaking changes in addon modules like Sitemap or AtomFeed.

This also means that it should be easier for new contributors to find a package and make contributions, since there will be less far-reaching consequences. So if anyone sees one of these new packages and would like to take a more active role (either contributing, or even taking over maintainership), please let me know.

Anyway, on to the topic at hand: the magic.

## Static subsite

There are two pieces of "magic" involved in the static subsite (for serving static files). The first is the very concept of a subsite. Unfortunately, this is a topic that I have not gotten around to documenting properly, and probably will not for a few months. For a one line explanation: it allows you to create reusable components that can be plugged into any Yesod application.

But the magic to be addressed now is the staticFiles function. Let's review a few key facts about how the static subsite works:

* There is a StaticRoute datatype which represents each file request. There is a single constructor (StaticRoute) which takes two arguments: a list of path pieces (eg, "/foo/bar.css" becomes ["foo", "bar.css"]) and a list of query string parameters.

* The Static datatype contains a field for the lookup function, which takes a FilePath. For this, the path pieces are converted back into a path (eg, ["foo", "bar.css"] -> "/foo/bar.css") and the query string is ignored.

So what's the point of the query string? If you place a hash of the file contents in the query string, then you can set you expires header far in the future and not worry about the client getting stale content. When you change the content, the hash will change, and therefore the URL will change, causing the browser to download a new copy.

We now have two annoyances when dealing with the static subsite:

* We need to type in the file paths in our Haskell code without any assurances that the files actually exist. You're one typo away from a broken site.

* You'll need to manually calculate the hashes. Besides the extra programming overhead, if you do this at runtime you'll get a major performance overhead as well.

The answer to both of these is the staticFiles TH function. If you give it a filesystem path, it will explore the entire directory structure, find all files, calculate hashes, and create Haskell identifiers for each of them. And it does all of this at compile time, meaning zero runtime performance overhead. So, if you have a "static/images/logo.png" file, and you want to use it, you simply include the line:

    $(staticFiles "static")

in your code and you will now have a <code>images_logo_png</code> value in scope with a datatype of <code>StaticRoute</code>. Oh, and I forgot to mention: GHC 6.12 introduced a feature where the TH brackets are not necessary for top-level calls, so you can simply write

    staticFiles "static"

There is one downside to this approach that needs to be mentioned: if you change files in your static folder without modifying the module that calls staticFiles, you will still have the old identifiers in your object files. I recommend having a StaticFiles module in each project that just runs the staticFiles function. Whenever you modify your static folder, <code>touch StaticFiles.hs</code> and you should be good to go. For extra safety when compiling your production code, I recommend always starting with a cabal clean.

## parseRoutes

The parseRoutes quasi-quoter is actually even simpler than Julius. However, it goes hand-in-hand with mkYesod, which is significantly more sophisticated than Julius, and therefore this section ended up in this post instead. The quasi-quoter does only two things:

* Converts each non-blank line in its argument into a [Resource](http://hackage.haskell.org/packages/archive/web-routes-quasi/0.6.2/doc/html/Web-Routes-Quasi-Parse.html#t:Resource).

* Checks that there are no overlapping paths in the resources provided.

Starting with the second point, an overlapping set of paths could be something like:

    /foo/#String
    /#String/bar

since /foo/bar will match *both* of those. However, there is unfortunately a little bit more to it than that, since even these paths will overlap:

    /foo/#Int
    /#Int/bar

This is because the quasi-quoter doesn't know *anything* about what an Int or String are, it just passes them along. I still think that it is best to avoid such overlapping paths, but if you *really* want to avoid the overlapping check, you can use [parseRoutesNoCheck](http://hackage.haskell.org/packages/archive/web-routes-quasi/0.6.2/doc/html/Web-Routes-Quasi-Parse.html#v:parseRoutesNoCheck).

Now what about that Resource datatype? It has a single constructor:

    Resource String [Piece] [String]

The first String is the name of the resource pattern, and the list of Strings at the end is the extra arguments. For example, in:

    /foo/bar FooBarR GET POST

that list of Strings would be

    ["GET", "POST"]

The quasi-quoter does not apply any meaning to that section; that is handled by mkYesod. As far as the list of Pieces, there are three piece constructors: StaticPiece, SinglePiece and MultiPiece. As a simple example,

    /foo/#Bar/*Baz

becomes

    [StaticPiece "foo", SinglePiece "Bar", MultiPiece "Baz"]

Of all the magic in Yesod, this is the part that can most easily be replaced with plain Haskell. In fact, this could be a good candidate for an IsString instance. Something to consider...

## mkYesod

I personally think that type safe URLs are the most important part of Yesod. I feel *very* good saying that, because I'm not even the one who came up with the idea: after release 0.2 of Yesod (I think), both Chris Eidhof and Jeremy Shaw emailed me the idea. It's actually hard for me to imagine where Yesod would have gone had it not been for that recommendation.

The good side of type safe URLs is that it makes it all but impossible to generate invalid internal links, it validates incoming input from the requested path, and makes routing very transparent. The bad side is that it requires a *lot* of boilerplate:

* Define a datatype with a constructor for each resource pattern.
* Create a URL rendering function to convert that datatype to a [String]
* Write a URL parsing function to convert a [String] to that datatype (well, wrapped in a Maybe)
* Write a dispatch function to call appropriate handler functions.

mkYesod is probably the most important single function in all of Yesod. It does all four of these steps automatically for you, based on a list of Resources (which can be created using the parseRoutes quasiquoter described above).

As a simple example, let's take a look at what mkYesod does:

    {-# LANGUAGE QuasiQuotes, TypeFamilies #-}
    import Yesod
    import Yesod.Helpers.Static

    mkYesod "MySite" [$parseRoutes|
    / RootR GET
    /person/#String PersonR GET POST
    /fibs/#Int FibsR GET
    /wiki/*Strings WikiR
    /static StaticR Static getStatic
    |]

If you run this code with -ddump-splices, you'll see the resulting Haskell code. Here's the cleaned up version:

    data MySiteRoute
        = RootR
        | PersonR String
        | FibsR Int
        | WikiR Strings
        | StaticR Route Static
        deriving (Show, Read, Eq)

    type instance Route MySite = MySiteRoute

    dispatch RootR method =
        case method of
            "GET" -> Just $ chooseRep <$> getRootR
            _ -> Nothing
    dispatch (PersonR x) method =
        case method of
            "GET" -> Just $ chooseRep <$> getPersonR x
            "POST" -> Just $ chooseRep <$> postPersonR x
            _ -> Nothing
    dispatch (FibsR x) method =
        case method of
            "GET" -> Just $ chooseRep <$> getFibsR x
            _ -> Nothing
    dispatch (WikiR x) _ = Just $ chooseRep <$> handleWikiR x
    -- Yes, this next bit is *ugly*...
    dispatch (StaticR x) method =
        (fmap chooseRep <$>
            (toMasterHandlerDyn
                StaticR
                (\ -> runSubsiteGetter getStatic)
                x
                <$>
                Web.Routes.Site.handleSite
                (getSubSite :: Web.Routes.Site.Site (Route Static) (String -> Maybe (GHandler Static MySite ChooseRep)))
                (error "Cannot use subsite render function")
                x
                method))

    -- produces a pair of path pieces and query string parameters
    render RootR = ([], [])
    render (PersonR x) = (["person", toSinglePiece x], [])
    render (FibsR x) = (["fibs", toSinglePiece x], [])
    render (WikiR x) = ("wiki" : toMultiPiece x, [])
    render (StaticR x) =
        (\ (b, c) -> (("static" : b), c)) $
        (Web.Routes.Site.formatPathSegments
            (getSubSite :: Web.Routes.Site.Site
                (Route Static)
                (String -> Maybe (GHandler Static MySite ChooseRep))) x)

    parse [] = Right RootR
    parse ["person", s] =
        case fromSinglePiece s of
            Left e -> Left e
            Right x -> PersonR x
    parse ["fibs", s] =
        case fromSinglePiece s of
            Left e -> Left e
            Right x -> FibsR x
    parse ("wiki" : s) =
        case fromMultiPiece s of
            Left e -> Left e
            Right x -> WikiR x
    parse ("static" : s) =
        case Web.Routes.Site.parsePathSegments
           $ (getSubSite :: Web.Routes.Site.Site
                (Route Static)
                (String -> Maybe (GHandler Static MySite ChooseRep))) of
            Left e -> Left e
            Right x -> StaticR x
    parse _ = Left "Invalid URL"

    instance YesodSite MySite where
        getSite = Web.Routes.Site.Site dispatch render parse

The actual code is a little bit harder to follow, but does the same basic thing. One last thing: in order to make it possible to define your routes in one file and your handlers in a bunch of other files, we need to split up the declaration of the MySiteRoute datatype from the declaration of the dispatch function. That's precisely the purpose of providing both [mkYesodData](http://hackage.haskell.org/packages/archive/yesod/0.6.7/doc/html/Yesod-Dispatch.html#v:mkYesodData) and [mkYesodDispatch](http://hackage.haskell.org/packages/archive/yesod/0.6.7/doc/html/Yesod-Dispatch.html#v:mkYesodDispatch).

## Errata for last post

One thing I forgot to mention in the last post: Hamlet templates are in fact polymorphic. You can have:

    [$hamlet|%h1 HELLO WORLD|] :: Html
    [$hamlet|%h1 HELLO WORLD|] :: Hamlet a
    [$hamlet|%h1 HELLO WORLD|] :: GWidget sub master ()

This is achieved via the [HamletValue](http://hackage.haskell.org/packages/archive/hamlet/0.6.1.2/doc/html/Text-Hamlet.html#t:HamletValue) typeclass. This construct is a little complicated, and probably deserves its own discussion. For now, I will simply say that this typeclass provides htmlToHamletMonad and urlToHamletMonad functions for the hamlet TH code to call, and thus create a polymorphic result.

There are two important things to keep in mind:

* You cannot embed a template with one datatype inside a template with a different datatype. For example, the following will not work:

    asHtml :: Html
    asHtml = [$hamlet|%p This is just Html|]
    asHamlet :: Hamlet MySiteRoute
    asHamlet = [$hamlet|^asHtml^|]

* When dealing with the GWidget instance, GHC can get confused. For example:

    -- this works
    myGoodWidget :: GWidget sub master ()
    myGoodWidget = do
        setTitle "something"
        [$hamlet|%h1 Text|]

    -- this doesn't
    myBadWidget :: GWidget sub master ()
    myBadWidget = do
        [$hamlet|%h1 Text|]
        setTitle "something"

Since the datatype for the hamlet quasiquotation in myGoodWidget is required to be GWidget sub master (), everything works out. However, in myBadWidget, the datatype is actually GWidget sub master __a__, and GHC doesn't know that you want to use the GWidget sub master () instance of HamletValue. The trick to get around this is to use the addWidget function, which is:

    addWidget :: GWidget sub master () -> GWidget sub master ()
    addWidget = id

## Conclusion

I still owe you another post persistent entity declarations and migrations, but that will have to wait for another day. I still have some coding to do tonight!
