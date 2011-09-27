I don't normally make release announcements for minor release (ie, the third number in the version). But just because [0.6.3](http://hackage.haskell.org/package/yesod-0.6.3) is introducing some cool stuff, I thought I'd mention it here. And while I'm at it, I'll cover all the new features since 0.6.0 was released. In no particular order:

* Matt Brown has added some better subsite support. In particular, the YesodSubRoute typeclass, addSubWidget and toMasterHandlerMaybe. Unfortunately, due to a bug in GHC 6.12, his more impressive patches can't be included yet.

* Full GHC 7 support. GHC 7 changes quasi-quotation syntax to no longer use the dollar sign. Using some CPP hackery, Yesod will now compile on both 6.12 and 7. The scaffolded site also adapts appropriately.

* The scaffolded site minifies Javascript using Alan Zimmerman's [hjsmin](http://hackage.haskell.org/package/hjsmin). Both CSS and Javascript results are now only written to disk once.

* The scaffolded site includes a widgetFile function. If you include $(widgetFile "name"), it will automatically include the Hamlet, Cassius and Julius templates named "name" if available.

* runFormTable and runFormDivs provide some nicely packaged support for form submissions. It takes care of all the HTML and CSRF protection automatically.

* Fixed some minor bugs in joinPath/splitPath affecting corner cases. Also fixed a cookie expire bug that had cookies expiring at the end of each session. (By the way, that's why Haskellers would always log you out immediately. That's been fixed now.)

* fileField, maybeFileField, and atomLink function.

* Support for blaze-builder 0.2 and network 2.3.

Please check it out, and let me hear your thoughts. I hope the moves we're making in these releases are helping to solidify the API and fill in the gaps. I'm still up in the air about whether there will be a 0.7 release or if we'll skip straight to 1.0.

One last comment: the book has had a **massive** internal restructuring to convert it to an XML format. I've done this for a number of reasons, but more importantly for you, this means I will more easily be able to generate PDF versions. I'm still in the process of converting some of the old markdown to the newer XML, but hopefully the process won't take too long. If you notice any bugs, please let me know.
