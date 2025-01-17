## Yesod Documentation Efforts

We love the new Yesod documentation system! We can use Markdown or a WYSIWYG editor, depending on whatever is convenient. We can easily take content from a blog post or a mail list response and put it in the [book](/book) or the [wiki](/wiki). Speaking of which, the wiki doesn't suck anymore! I have been using it and working out some kinks with Michael, and it is ready for contributors now. And we still support comments- thanks to those that told us about problems with the Yesod examples in the book, which are now fixed.

Our biggest documentation hassle now is that our haddocks frequently don't build (due to dependency haddocks not building). We hope the haskell community can fix this on the current hackage server. Otherwise we are forced into the hassle and user confusion of hosting our own haddocks.


## Yesod Wiki is open sourced!

Michael's employer, Suite Solutions, not only sponsored the initial work on what we are using for the Yesod Wiki, but has now decided to [open source it!](https://github.com/snoyberg/yesodwiki)

## Lets keep talking about wikis - haskellwiki.org

Haskellwiki is slow! I found it a pain to work with because edits would frequently take long periods or timeout. Simple page requests often take a long time. My guess as to the root of the problem is that either the database is overwhelmed, or that PHP's blocking IO means that requests are being delayed. If the latter, this is a problem that Haskell has solved in the runtime! In most languages, like PHP, unless one uses a specific asynchronous driver, an IO request (to the database) will block the current process. The only thing that can be done is to spin up another process, which ends up bloating memory. In haskell, the current thread will block until the database call returns, but another thread can immediately take on the next request. And we don't even need any special libraries in Haskell! Not only that, but we can deploy a single binary that can spread across multi-core, rather than managing separate processes.

Of course the easiest solution to haskellwiki probably lies in fixing the php deployment- maybe we can get someone with MediaWiki experience to speed the wiki. But shouldn't the wiki for Haskell be written in Haskell? Starting a new haskellwiki written in Haskell wouldn't be a problem. The big pain is converting all the existing MediaWiki content. Pandoc already has a MediaWiki reader, but it is probably incomplete and there are still going to be files to transfer. And maybe user accounts also. It would be great to see someone step up and take on this task. And [John MacFarlane would love to see a new maintainer of gitit](http://www.mail-archive.com/haskell-cafe@haskell.org/msg90810.html).


## Keep up the Docs

Lets not let our documentation infrastructure decay and make documentation more of a hassle. The problem with maintaining community infrastructure is probably that it is a thankless job. Can we have a donation drive that encourages every Haskell user to donate $20 to improve haskell infrastructure? That way we could at least thank some of the maintainers with money.

Thanks to everyone who documents their libraries, writes tutorials, and answers questions on mail lists- your contributions make the Haskell community work!