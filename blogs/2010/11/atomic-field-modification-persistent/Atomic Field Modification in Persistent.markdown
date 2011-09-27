I'm happy to announce version 0.3.1 of [persistent](http://hackage.haskell.org/package/persistent). Liam O'Connor [pointed out to me](https://github.com/snoyberg/yesod/issues/#issue/14) that not only was there a huge performance gap in the previous method of doing mass modifications to values in persistent, but we were losing all of the wonderful ACIDity offered by our database.

The idea is pretty straight-forward: let's say you've got a Person entity:

    Person
        name String
        age Int Add

That Add means it's now possible to do something like:

    update personId [PersonAgeAdd 1]

to make someone 1 year older. Currently, the four verbs I've added are Add, Subtract, Multiply and Divide, though in theory others could be added as well. Let me know if there's demand.

## Naming cleanup

One other thing. Let's review some of the words that we use in persistent:

* Asc
* Desc
* Eq
* Lt
* Add
* Divide
* update
* null

Hmm... something doesn't quite fit in here. To make things more consistent, I'm deprecating the last two. In the future, you should use Update instead of update, and Maybe instead of null. Besides the capitalization consistency, semantically null was the wrong word here: I tried very hard to make sure the behavior of a "nullable" field would closely match a Maybe value and **not** the hodge-podge of NULLity which exists in SQL. Plus, Persistent has nothing to do with SQL (in theory at least), so we shouldn't be using SQL-specific names.

For now, this will just print out a message during compile time that the names are deprecated. I'll eventually remove support for them entirely, but I'm not in any rush.

## Minor Yesod release

Greg Weber keeps insisting that I make Yesod better, so I announce version 0.6.2. This is a minor release, only introducing two new functions: runFormTable and runFormDivs. These functions make it just slightly easier to have properly CSRF-protected forms, by building the whole widget up for you. No need to fiddle with %form!method=POST!action=@something@, this does it.

On a more general note, there's a problem of increased complexity due to polymorphic forms. In particular, error messages can get out of hand. I'm aware of the issue, and I'm trying to figure out a way to solve it. If anyone has any ideas, let me know. One thought that keeps coming to mind is removing the GHandler that's at the base of the GForm monad stack. My reasoning is that I don't think anyone is actually using the power, and it might (I repeat: **might**) simplify things.

If anyone is actually utilizing this feature as-is, please let me know. Basically, it allows you to perform arbitrary functions- such as database lookup- when constructing forms. This in theory is very cool: you can look up a list of authors to select from when designing a library book entry site, for instance. In practice, it's tedious to write code this way, and it can all be done more easily with a parameter to the formlet function with the list of authors.

Anyway, I'm still just at the brainstorming phase of this. Input is greatly appreciated.
