Well, Yesod 0.7 is almost ready for release. Thanks to everyone for some great ideas: I consider the "Please Break Yesod" campaign to be a huge success. There have been a huge number of API tweaks, feature enhancements, and all around goodness.

I have just converted the [Haskellers](http://www.haskellers.com) codebase to Yesod 0.7, and have taken detailed notes on changes you'll need to make. The process is not that bad really: it took about half an hour for me to convert the code __and__ write up the docs.

So that we can get inline commenting, I've put the guide up as [a chapter of the Yesod book](http://docs.yesodweb.com/book/migrate6to7/). I'm planning on making the actual release Saturday night Israel time, assuming I don't get any major bug reports before then.

I will hold off on the list of new features for the release announcement. However, I *have* just released some new packages: Hamlet 0.7, Persistent 0.4 and wai-handler-devel 0.2. Some quick comments:

* Hamlet 0.7 involves a lot of new syntax changes. I appreciate the community's feedback on this: I think we've created a much better product. Check out [my previous blog post](http://docs.yesodweb.com/blog/hamlet6to7/) for migration information, and the [Wiki page](http://wiki.yesodweb.com/Hamlet%200.7%20changes) for some of the surrounding discussion.

* Persistent saw no real changes. The main reason for the new major version number is the migration to the [monad-peel package](http://hackage.haskell.org/package/monad-peel). By the way, that package is a work of art, I strongly recommend people start looking into it as a replacement for MonadCatchIO-*. Anders has done an amazing job.

* wai-handler-devel has a much more elegant underlying approach (which if I had time I would blog about). It's actually getting close to a proper plugin system, which is something I'm interested for my upcoming LambdaEngine project. &lt;/tease&gt;

## New site design

I've also finally revamped the Yesod site to use the new Haskell color theme that's been going around. I personally like the change, but wouldn't mind any critiques. We also have a new Yesod logo. If it looks a little funny to you, flip it upside down, it should look more familiar that way.

## Yackage

For those of you who can't wait, or who want to help me out with some testing, you can get the most current version of the packages from [Yackage](http://yackage.yesodweb.com/). If you find any bugs, please let me know!
