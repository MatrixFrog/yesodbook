I've just released new versions of Hamlet and yesod-core which provide the foundation for a complete i18n solution in Yesod. This post is intended to give an understanding of the details of this implementation. Towards the end, I'll discuss the higher-level goals that are intended for day-to-day usage. On the way, we're going to make a stop off and discuss polymorphism, widgets and type-safe URLs.

Side note: I've also just release a new version of Yesod (0.8.1), which in addition to the i18n stuff described below also provides a (hopefully) working "yesod devel". If people are still running into trouble, please let me know.

## I18N: Datatypes and functions

I've spent a lot of time thinking about what the ideal approach to I18N is. Without getting into comparisons with other solutions (I'm sure there will be plenty of time for that later), I think that simple ADTs are in fact a nearly perfect solution. Let's say we've got an app that needs to tell the user hello, the date and how many books the user has just purchased. Let's go through the issues:

* For hello, we simply need a way to provide strings in different languages.
* The date display needs to be localized based on language/locale. For en_US, we would likely want month/day/year, whereas for de_DE, we would want day/month/year.
* The doozy is the books. We need to address:
    * Word order. For example, in most languages, the number precedes the noun. In Hebrew, this is true *unless* you're dealing with the number one, in which case it follows the noun.
    * Pluralization. Don't even think about saying "you have purchased 3 book(s)." Oh, and by the way, Russian is really tricky.
    * Gender. English may only have one version of "seven", but many other languages do not.

I think we can model an *incredibly* simple solution to this in Haskell. Let's start off with our datatype:

    data Message = Hello    
                 | TodayIs Day
                 | BooksPurchased Int

I think we can agree that this datatype will represent all the information necessary for a native speaker of a language to formulate the correct output. In fact, a native speaker could likely write a function to do this automatically. For example:

    english :: Message -> String
    english Hello = "Hello"
    english (TodayIs day) = "Today is " ++ formatTime defaultTimeLocale "%m/%d/%Y" day
    english (BooksPurchased num) = concat
        [ "You have purchased "
        , show num
        , pluralize num "book" "books"
        ]

    pluralize 1 x _ = x
    pluralize _ _ y = y

So as to not embarrass myself and my rudimentary high school Spanish, I'll avoid writing similar functions for other languages, but you can see that it's possible to write a similar spanish, french, german and russian function. And you can go as crazy with grammar rules as you want, since you have the full expressivity of Haskell at your disposal.

So assuming we have all those functions, let's say we want to write a web app that will display the page in the correct manner based on the user's accept-language header. This header actually gives a prioritized list of languages. So we could write a function like so:

    type Lang = String
    translate :: [Lang] -> Message -> String
    translate ("en":_) = english
    translate ("es":_) = spanish
    translate ("de":_) = german
    translate (_:rest) = translate rest
    translate []       = english -- The default backup

### YesodMessage

And introduced in yesod-core 0.8.1 is the new YesodMessage typeclass. It is built on the ideas above, with a few modifications:

* We want different applications to provide their own translations of messages, so the type class applies to the foundation datatype, not the message itself.
* The message is provided via an associated type (i.e., type family).
* We use both the subsite and master site foundation datatypes in this typeclass, so that you can have a different translation for a combination. This allows the new yesod-auth, for example, to let app writers provide their own translation strings.
* We use Text instead of String for the language.
* And we use Html for the output.

The definition of the typeclass is very simple:

    class YesodMessage sub master where
        type Message sub master
        renderMessage :: sub
                      -> master
                      -> [Text] -- ^ languages
                      -> Message sub master
                      -> Html

There's also a helper function:

    getMessageRender :: (Monad mo, YesodMessage s m) => GGHandler s m mo (Message s m -> Html)

This can be used as follows:

    getRootR = do
        render <- getMessageRender
        defaultLayout $ do
            setTitle $ render Welcome
            addWidget [hamlet|
    <p>#{render IntroLine}
    <h6>#{render Copyright}
    |]

As you can see, this system works very nicely, but can be a bit tedious. We'll get back to the tediousness a little later.

## Html and Hamlet

Let's take a break and talk about datatypes in Hamlet. The first datatype is Html. This is provided by the blaze-html package, and represents content that can be immediately rendered to a bytestring and sent to a client to view. Easy enough.

However, we also have the Hamlet datatype. This is defined as:

    type Render url = url -> [(Text, Text)] -> Text
    type Hamlet url = Render url -> Html

This type signature is the basis of type-safe URLs. A "Render url" is a function that can convert a type-safe URL and a query string into a textual representation that can actually be sent to the client. For example, with a URL datatype of:

    data MyRoute = Home | Blog

we could define a render function (ignoring query strings for now) as:

    renderRoute Home = "http://www.mysite.com/"
    renderRoute Blog = "http://www.mysite.com/blog"

So now we can build up a Hamlet value using these datatypes:

    myHamlet = [hamlet|<a href=@{Blog}>Visit my blog!|]

and using our render function get a plain old Html value:

    myHtml = myHamlet renderRoute

### Why bother?

But what's the point of this complication? We could just as easily write the above as:

    myHtml = [hamlet|<a href=#{renderRoute Blog}>Visit my blog!|]

Well, there's a few advantages of the first approach:

* It avoids the need to explicitly pass the renderRoute function around.
* You can easily swap out different rendering functions. One real life example of this would be to generate relative URLs instead of absolute: the render function from /foo/bar will be different than the render function from /foo/bar/baz.
* You can't accidently mess up datatypes.

But if you notice, there's a very strong similarity between this second approach and the initial i18n example of Hamlet shown above.

## New Hamlet interpolation

So now I'd like to borrow an idea from type-safe URLs. Just like a URL gets a special interpolation in Hamlet to avoid tediousness and keep type safety, let's do the same thing for i18n. Since the underscore is used so prevalently for this in existing applications, we'll use it as well. So starting with Hamlet 0.8.1, we can write our i18n version of getRootR above like this:

    getRootR = do
        render <- getMessageRender
        defaultLayout $ do
            setTitle $ render Welcome
            addWidget [hamlet|
    <p>_{IntroLine}
    <h6>_{Copyright}
    |]

We still need to get the explicit message renderer for the setTitle call, since it exists outside of Hamlet. But within Hamlet, we get a nice interpolation syntax. Using this, we could write our little bookstore application very easily:

    postBuyBooksR = do
        count <- processOrder
        defaultLayout $ do
            addHamlet [hamlet|
    <p>_{BookPurchased count}
    |]

But I've lied a little bit: the code above won't actually work. The reason is that the Hamlet datatype doesn't know anything about messages, it only knows about type-safe URLs. But the approach is basically identical: we need to provide a function that renders some datatype into some value that Hamlet knows how to embed. So Hamlet 0.8.1 introduces a new type synonym for __I__nternationalized Hamlet:

    type Translate msg = msg -> Html
    type IHamlet msg url = Translate msg -> Render url -> Html

Hopefully, the pattern is becoming clearer: whenever we have a special datatype, we need to provide a rendering function.

## Polymorphism

Something that has confused a lot of people is polymorphism in Hamlet. It's a very nifty feature which allows a Hamlet template to have a few different types: Html, Hamlet or even a Widget. However, it comes with a lot of baggage:

* Its implementation is relatively slow.
* It can produce ridiculously complicated error messages.
* Polymorphism only goes so far.

The last point refers to the fact that template interpolation (^{...}) is actually *not* polymorphic. Let me explain with some code. What is the type of:

    [hamlet|#{someText}|]

? Well, whatever you want it to be: Html, Hamlet, or Widget. But what about:

    [hamlet|^{someOtherTemplate}|]

? It's constrained by the type of someOtherTemplate. So as convenient as it is to just have one quasi-quoter (hamlet) and one template Haskell function (hamletFile) that works for multiple datatypes, it's just not worth it. Hamlet 0.8.1 includes a new module, Text.Hamlet.NonPoly. In this module, the hamlet quasiquoter will only ever produce a Hamlet value. If you want an Html value instead, use the html quasiquoter. And if you want an IHamlet, you can use the ihamlet quasi-quoter. So for example:

    getRootR = do
        mrender <- getMessageRender
        urender <- getUrlRender
        defaultLayout $ do
            setTitle $ mrender Welcome
            addHtml $ [ihamlet|
    <p>_{IntroLine}
    <h6>_{Copyright}
    |] mrender urender

But to keep up with the crowd, yesod-core has added its own variant of this family of functions: whamlet. So the code above can be written even better as:

    getRootR = do
        render <- getMessageRender
        defaultLayout $ do
            setTitle $ render Welcome
            [whamlet|
    <p>_{IntroLine}
    <h6>_{Copyright}
    |]

Notice a few things:

* We no longer use addHtml, since the result of whamlet is a Widget.
* We don't need to pass the render function to the template; Yesod does that automatically.

To keep backwards compatibility, the polymorphic functions will remain standard until the 0.9 release, at which point they will be removed and replaced with the non-polymoprhic versions. (Most likely, we'll be removing the hamletFileDebug and runtime Hamlet code at the same time, unless I hear a good argument for keeping them.) And in addition to the quasi-quoters, there are external file versions of all these systems (htmlFile, hamletFile, ihamletFile and whamletFile).

And as a total aside: this assault on polymorphism is a bit of a theme in the 0.9 release. We're also planning a revamp of the yesod-form package to do away with its polymorphic abilities as well.

## High level interface

I think the interface this provides for the template writer is already very nice: the vast majority of the time, a template author will just use the combination of whamlet(File) and _{Message} and be done with it. The thing I'm concerned about is how we're going to define YesodMessage. Clearly we don't want translators to need to learn Haskell. So we need some new file format that they'll be comfortable with.

Here's where I would really appreciate input. I'm currently thinking of having two types of files: one defining the messages, and one giving translations. We could use a similar syntax to Persistent for the former:

    Hello
    TodayIs
        day Day
    BooksPurchased
        books Int

which would become:

    data Message = Hello | TodayIs { day :: Day } | BooksPurchased { books :: Int }

We would then have files for each language that look like:

    Hello: Hello
    TodayIs day: Today is #{monthYearDay day}
    BooksPurchased books: You have purchased #{show books} #{pluralize "book" "books"}

We'll stick all of these into a single folder. Then in our Haskell file, we'll write something like:

    $(messages "folder" settings)

The settings could contains stuff like language aliases (x_Y is served x is there is no specific x_Y, for instance) and the default language. The system can then make the following guarantees:

* The translator is using the correct variable name. If he/she typed in "BooksPurchased day", the system can give an error- and at compile time no less.
* The translator is using the correct number of variables. "BooksPurchased day books" will also break.
* The default language has translations available for all strings. That can be optional for other languages.

The other thing we'll need is a library of helper functions. You've seen two already: monthYearDay and pluralize. I won't speculate too much on which other functions such a library would require, I'd rather let it grow naturally.

I think the best way to approach this problem is to actually translate a real site. So after getting the initial infrustructure in place, my next goal will be to get it running in the context of Haskellers. If anyone knows some non-English languages and would be interested in helping out with this, please let me know. And in general, I'm interested to hear feedback on the ideas here, especially the critical type.
