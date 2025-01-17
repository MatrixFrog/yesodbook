Haskell is an amazing programming language that has helped us create an amazing web framework. We leverage Haskell's capabilities wherever we can, but we want to go over some of its biggest weaknesses. In this post, any mention of Haskell really means Haskell using the GHC compiler.

Yesod is in a somewhat unique position of being a highly used Haskell project with many dependencies. And we haven't just created a web framework- we have created a new set of template languages and Haskell's first "ORM". We are pushing the boundaries of the language in every area, and mostly finding that Haskell was well designed for all of this. There are certainly more complaints that could be make about Haskell, but we want to focus on practical issues that we have found in Yesod that are the most difficult to workaround.

## Limitations with unknown solutions

### Error messages from Complex types

In Yesod we have some complex types in a few key places. And we are fine with having complex types internally. However, exposing them in an interface is a very delicate matter- users have to be able to understand error messages. Yesod has abandoned certain uses of polymorphism due to this issue. And we would still like to make error messages better.

Reading error messages is an interface through which programmers learn Haskell. Is there a way to make error messages better? The most interesting approach I can think of is to somehow let the library writer specify certain error messages.


## Limitations with partial solutions

### no stack traces

This is less of a bother than one would think when coming from another language, but still a problem. Simon Marlow recently stated he may be able to come up with a solution, which was very exciting news. Yesod now supports logging statements that contain file and line number information using Template Haskell. I released a [package](http://hackage.haskell.org/package/file-location) that applies the same technique. This allows the programmer to record the file and line number location, but not a stack trace. There is also a [package](http://hackage.haskell.org/cgi-bin/hackage-scripts/package/monadloc) that is designed to give a stack trace for an exception.


### code reloading

Invoking `yesod devel` will automatically recompile a Yesod project with cabal as files change, and start the new program in place of the old. However, the re-linking time dominates this and makes for slower reloading that we would like. There is also a plugins package for Haskell for re-loading the code that Happstack has recently been improving. And Snap has some techniques for re-loading Handlers.

We would really like what some dynamic languages have achieved- only code relevant to what has changed get re-interpreted, and this is done quickly and relatively reliably. But between the various Haskell web frameworks it seems that we are coming up with decent solutions.



## Limitations that can be solved now

### Dependency Hell and using unreleased (beta or internal) code.

Dependency Hell is when you type `cabal update && cabal install` and cabal informs you it can't install the packages, even though your package specifications are correct.

Ever since the Bundler Ruby tool matured, I have never once been on a Ruby project where the libraries could not be immediately installed. This is what Haskell is competing against, and we shouldn't settle for less.

cabal-dev has been a great stride towards solving these dependency issues, and I feel like Haskell is  getting closer to the goal of flawless installs.

The other issue with cabal is specifying using code that isn't on Hackage. I should be able to specify in a cabal file to use a package version that is on the file system or under version control on github. This makes beta releasing dead simple- just have users to edit their cabal file. It also makes having local changes very simple- just point the package in the cabal file to your local file system.

There is also a [new attempt to give Haskell a more reliable interface than just version numbers](http://skilpat.tumblr.com/post/9411500320/a-modular-package-language-for-haskell). I think of this as focused on preventing installations that fail to compile, whereas I view dependency hell as mostly being about failing to configure and start the installation.


### Template Haskell re-loading

Template Haskell (TH) that performs IO does not automatically recompile itself when it should. In Yesod we have proved the value of having compile-time templates in external files. This puts Haskell on par with dynamic languages in terms of ease of using templates. It actually puts Haskell in a much better position because the templates are evaluated for errors at compile time. But when an external hamlet template changes, it won't automatically get recompiled by ghc. We are able to work around this in our development environment by watching for changes to external templates. However, this is a frail solution, and we believe that GHC should provide this capability for us. We want to have a function loadQQ, which takes a quasi-quoter and a file path. loadQQ will automatically recompile the hs file when the quasi-quoted file has changed.

### Namespace clashing, particularly for record fields

~~~~~~~~~{.haskell}
data Record = Record { a :: String }
data RecordClash = RecordClash { a :: String }
~~~~~~~~~

Compiling this file results in:

~~~~~~~~~~~~
record.hs:2:34:
    Multiple declarations of `Main.a'
    Declared at: record.hs:1:24
                 record.hs:2:34
~~~~~~~~~~~~~

In the Persistent data store library, we work around the issue by having the standard of prefixing every record field with the record name (recordA and recordClashA). But besides being extremely verbose, it also limits us from experimenting with more advanced features like a partial record projection or an unsaved and saved record type.

The verbose name-spacing required is an in-your-face, glaring weakness telling you there is something wrong with Haskell. This issue has been solved in almost every modern programming languages, and there are plenty of possible solutions available to Haskell.

[Here is one](http://hackage.haskell.org/trac/haskell-prime/wiki/TypeDirectedNameResolution) solution. The Haskell community is generally wonderful at advancing the language (or at least advancing GHC). Records is one place where this process has completely fallen apart, and Haskell has become a backwards language.

So we want to start a discussion: how can we solve Haskell name-spacing, at least for records? From Yesod's perspective we don't care what the solution is, just as long as we have one. Let us know what your thoughts are here, or on Reddit, and we are going to take this to GHC HQ and figure out a way to finally solve this issue.


## Lets make Haskell better!

I want to stress that Haskell is an amazing language that solves many problems that other languages will be eternally stuck with. The great thing is that all the problems listed here can eventually be solved. We love using Haskell for Yesod. We just want to make it even better. 