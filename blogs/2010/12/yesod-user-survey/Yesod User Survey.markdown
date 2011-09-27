I've been meaning to do this for about a month now, and I finally took the time to write it up: [the first Yesod user survey](https://spreadsheets.google.com/viewform?formkey=dDRhcFVCRVdjbXZuNXJGVUItb3lsQ0E6MQ). My goal is to get a good feel of what users like and don't like, and to know where to target efforts going forward. If you are a Yesod user, or even just *interested* in Yesod, please take the time to fill it out.

Also, a little status update on what's going on right now: there are a bunch of changes in the works for the next Yesod 0.7 release. Some things of note:

* A new version of WAI which will be faster and more general, allowing things like creating HTTP proxies.
* The new Hamlet syntax proposed by Greg. This will *not* deprecate the current syntax, though it may break some older Hamlet code if it involved a lot of copy-pasted HTML.
* Modularization: we're breaking things out into a **lot** of different packages.

And it's about this last point that I want to talk to you now. I'm trying to split out packages for two reasons: make core Yesod usage more lightweight, and to get some generic pieces of code available for usage outside of Yesod. As an example of the latter, I'm splitting out cookie parsing and resource pools.

I think this is a great opportunity for anyone who's been interested in Yesod to get their hands dirty. A lot of the work here is very minimal: pull out code, document it a bit, add some tests, and deploy to Hackage. If anyone is interested in helping out with this, please send me an email: I'll be more than happy to help out and explain the code, and I think it will make for a more robust community to share the knowledge out more.
