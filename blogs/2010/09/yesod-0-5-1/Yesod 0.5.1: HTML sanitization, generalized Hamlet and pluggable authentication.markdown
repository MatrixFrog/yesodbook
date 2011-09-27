For the past week or so, most of my programming focus has been going into projects tangential to Yesod development, mostly [http-enumerator](http://hackage.haskell.org/package/http-enumerator). As an aside: I think the [enumerator](http://hackage.haskell.org/package/enumerator) package is wonderful; does anyone want me to write up a tutorial on how to use it?

Anyway, there has been some activity on the Yesod side as well. If you remember, I'd previously mentioned that I thought Yesod was approaching a 1.0 release, marking it as more API-stable. I asked for some input on things people would like to see, and got some great responses. Here's a quick run-down of what's new. Fortunately, this release does not break any APIs, so you can easily upgrade.

### More comments

Normally I wouldn't mention updating documentation, but this is *slightly* different. I've now fully commented the site created by the yesod executable scaffolder. It should be easier to understand what's going on in there and why certain design decisions were made.

### HTML Sanitization

Greg Weber wrote the [xss-sanitize](http://hackage.haskell.org/package/xss-sanitize) package to remove dangerous bits from HTML. The HTML and NicHTML form fields now use this function to avoid any nefarious user input.

### Generalized Hamlet/Easier Widgets

Widgets make it easy to have modular components consisting of HTML, CSS and JavaScript. The downside is that it can be a bit tedious to use them. For example:

    myInnerWidget = do
        addBody [$hamlet|...|]
        addStyle [$cassius|...|]

    myOuterWidget = do
        inner <- extractBody myInnerWidget
        addBody [$hamlet|
            %h1 This is the inner widget
            ^inner^
        |]

Having to use the extractBody is a bit of an annoyance. Fortunately, this is no longer necessary. With the release of Hamlet 0.5.1, Hamlet templates are now generalized using the [HamletValue](http://hackage.haskell.org/packages/archive/hamlet/0.5.1/doc/html/Text-Hamlet.html#t:HamletValue) typeclass. This essentially means we can use Hamlet to produce things besides [Hamlet](http://hackage.haskell.org/packages/archive/hamlet/0.5.1/doc/html/Text-Hamlet.html#t:Hamlet) values.

On the Yesod side, we now have an instance of HamletValue for Widget. This means the code above can be written as:

    myInnerWidget = do
        addBody [$hamlet|...|]
        addStyle [$cassius|...|]

    myOuterWidget = [$hamlet|
        %h1 This is the inner widget
        inner <- extractBody myInnerWidget
        ^myInnerWidget^
    |]

Notice that we no longer prepend a call to addBody; the value of the quasi-quoted template is itself a widget. This also means that if we want to embed plain Hamlet templates, we'd need to do something like this:

    myPlainTemplate :: Hamlet MyRoute
    myPlainTemplate = [$hamlet|...|]

    myWidget = [$hamlet|
        %h1 Embed another widget
        ^myInnerWidget^
        %h1 Embed a Hamlet
        ^addBody.myPlainTemplate^
    |]

This may not seem like a big deal, but I think it will make people's coding much easier and allow us to write widget code in a more modular fashion.

## Upcoming features/Breaking changes

I mentioned previously that I was dissatisfied with the authentication subsite. In Yesod 0.6 I will be removing the Yesod.Helpers.Auth. Instead, I'm creating a separate [yesod-auth](http://hackage.haskell.org/package/yesod-auth) package. This will allow us to add dependencies to yesod-auth without affecting yesod, as well as make breaking changes without influencing yesod version numbers.

As I said at the beginning of this post, I've been working mainly on http-enumerator recently. The impetus to get started on that was to get good OpenID 2 support going for the haskellers.com website (nothing there yet). Currently, the Yesod authentication system only works with OpenID 1. I'm hoping that by Yesod 0.6, we'll have support for both of them.

## For some bright-eyed Haskeller out there...

One last feature I'd love to see implemented is a Javascript minifier to attach to Julius. Anyone out there want to take a swing at it?
