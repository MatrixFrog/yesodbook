I wanted to let everyone know that I've made some decisions about the future of
Hamlet: basically, I've decided to bite the bullet and make a bunch of breaking
changes at once to make Hamlet, Cassius and Julius more consistent and fit in
better with Haskell syntax. I'm still unclear on some minor details, so I will
hold off on a post explaining the changes until later.

However, I wanted to allay some fears: in case you are terrified at the thought
of having to convert all of your templates, worry not! This version of Hamlet
will include an executable called hamlet6to7 to automate the conversion. I've
already used it on the Yesod codebase, and everything seems to be working out
fine. Obviously, be smart when using this tool, and make sure you have your
code backed up (ie, already committed) when you run it.

The program takes a list of files to modify, and writes out a new file in the
same location. It modifies files with the file extensions hamlet, cassius,
julius or hs. For the first three, everything is pretty straight forward. For
Haskell source files, it makes modifications to quasi-quotation blocks. It
looks for a few different things:

* GHC 6.12 quasi-quotation ([$hamlet|...|])
* GHC 7 quasi-quotation ([hamlet|...|])
* CPP conditional code to select which version to run (#if GHC7...)
* CPP definitions to switch between 6.12/7 ([HAMLET|...|])

This should work for most everybody; if you find any bugs, let me know.

## Sample Hamlet

I'm actually quite happy with the new Hamlet syntax: I think it is much more readable and much closer to actual Haskell. Here is some sample code taken straight from Yesod:

    !!!

    <html>
        <head>
            <title>#{pageTitle p}
            ^{pageHead p}
        <body>
            $maybe msg <- mmsg
                <p .message>#{msg}
            ^{pageBody p}

## Lucius

One change I specifically did not make was switching Cassius to following Less
syntax. I *do* think this idea has merit, but instead of making a huge breaking
change for Cassius, I've deicded to add a *new* template language, Lucius (name
is up for debate btw), which will follow a different syntax. I can add this in
for Hamlet 0.7.1, and if everyone likes it, perhapsCassius will get deprecated.

## When's Yesod coming out?

I was actually hoping to have Yesod 0.7 available this week, but life got a
little crazy. Also, very recently, I started working on rewriting some major
pieces of Yesod's routing code. Previously, I had a huge mess of TH code,
bending over backwards to accomodate the web-routes Site datatype, and a very
weird split of code between Yesod and web-routes-quasi. I've made a few
decisions:

* web-routes is nice, but it doesn't fit Yesod properly.

* Dispatching and parsing need to happen simultaneously. This is to allow
  general WAI applications to be directly embedded as subsites.

* Wherever possible, combine the code for subsites and regular sites. I want to
  minimize the Template Haskell codebase as much as possible.

The impetus for this change was point number two: I believe we're going to be
seeing more WAI-specific applications in the future, and one goal I've always
had is great interoperability amonst Yesod web applications (that *is* the
reason for the WAI after all).

In any event, this code is now written, and fairly well tested. There's a few
datatype issues I need to figure out (does joinPath work on Strings of
ByteStrings, for example), and I need to update the scaffolder for recent
changes, and then we should be good to go.

With the new highly modularized breakdown of Yesod packages, we can easily
release an update to smaller components like yesod-form later on. So for the
moment, there have been no major improvements there, but they can be included
in point releases instead of requiring a brand new major release. An example of
such a change is migrating yesod-form to use digestive-functors in the future.
