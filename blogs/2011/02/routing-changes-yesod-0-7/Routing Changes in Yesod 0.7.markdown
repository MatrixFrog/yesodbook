One of the major changes in Yesod 0.7 is how routing works under the hood. From a user standpoint, not too much has changed, but it's important to understand to some extent how the new system works. This post will explain how Yesod *used* to work, why it changed, and describe two bugs that came up recently. Hopefully the bugs will give a deeper understanding of what the system is doing.

## Brief History Lesson

Back in the old days of Yesod 0.6 (you know, last week), routing consisted of:

* Convert a path info (eg, /foo/bar/baz/) into a list of strings (eg, ["foo", "bar", "baz", ""], notice the empty string). Thanks to Jeremy Shaw's excellent web-routes package for the utility functions for this.

* Apply cleanup rules: in order to ensure canonical URLs, we need to make sure to redirect non-canonical URLs to their canonical form. Though this was a user-controlled setting, the default settings ensured:

    * There are no double slashes.

    * If the last path segment had a period in it, it must not be followed by a trailing slash. (eg, /file.txt is good, /file.txt/ is bad).

    * Otherwise, there __must__ be a trailing slash (eg, /folder/ is good, /folder is bad).

* If the URL needed cleanup, send the user a 301. Otherwise, continue.

* Convert the path segments into a type-safe URL.

* Dispatch based on the type-safe URL value.

This seems to work out well, except for a specific use case: subsites. Point in case, let's say we have the static subsite. In an ideal world, the following should all happen:

1. /static/file.txt serves the file file.txt
2. /static/folder/ shows a directory listing for folder
3. /static/file_no_extension show file_no_extension
4. /static/file_no_extension/ redirects to /static/file_no_extension
5. /static/foo.folder/ shows a directory listing for foo.folder
6. /static/foo.folder redirects to /static/foo.folder/

However, since the cleanup rules are applied before dispatching to the subsite, we can only apply our default rules, which have no way of knowing that file.txt and file_no_extension should both be missing a trailing slash, while folder and foo.folder both __require__ a trailing slash.

## The new approach

The trick is we need to give subsites complete control over their route handling cleanup. Therefore, the new process goes something like this:

* Convert the path to pieces, like before.

* Attempt to dispatch.

* If dispatch fails, see if the route can be cleaned up at all. There are three possible results:

    * cleanPath returns a Left value. This means we should redirect the user to the given path.
    * cleanPath returns a Right value which is different than the current segments. This means that the URL could be cleaned up, but Yesod should not require a redirect, so we try dispatching on this new set of segments.
    * cleanPath returns a Right value which is the same as the current segments. In this case, dispatch simply won't work, and we should return a 404.

* The *new* default cleanup rules are much simpler: don't allow any empty path segments. This essentially means no double slashes, and no trailing slashes.

As you can see, this simplifies things greatly. Under this pattern, subsites like static are able to force their own canonical URLs, while the master site can still retain its version of cleanups.

## Bug 1: /book/

The first bug with the new system reared its head on this very website: going to http://docs.yesodweb.com/book/ would previously give the list of chapters in the Yesod book. With the new system, we no longer have the trailing slashes, so a request for /book/ should redirect to /book. Instead, the page was serving 404s.

However, a request to /about/ __did__ redirect to /about, so the problem was much more subtle than I'd initially imagined. It turns out that the following was happening:

* /book/ was translated to ["book", ""]
* ["book", ""] was compared against all of the routes. It did *not* match the BookR route, which would expected ["book"]. However, it *did* match ChapterR, which expects ["book", <any String>]. This is because the SinglePiece instance in web-routes-quasi matched empty strings.
* Yesod dispatched to the getChapterR function, giving it a parameter of "". However, there was no chapter named "", so getChapterR returned a 404.

The solution: release a new version of web-routes-quasi that does not match empty strings. Problem solved.

## Bug 2: can't force a trailing slash

I got a question from Dmitry Olshansky about restoring the previous set of URL cleanup rules. The basic approach is simple: write a custom cleanPath function in your Yesod typeclass which forces trailing slashes. For example (ignoring the period-in-last-piece issue):

    cleanPath _myapp pieces =
        if pieces == corrected
            then Right pieces
            else Left corrected
      where
        corrected = (filter (not . null) pieces) ++ [""]

However, when I tried this, all that happened was that no redirecting occurred: both a request for /foo/ and for /foo would return the same result.

The problem now is that dispatch happens before cleaning. /foo turns into ["foo"], which matches a resource, and therefore dispatch succeeds. /foo/ turns into ["foo", ""], cleanPath turns that into Right ["foo"], and the second phase of dispatch succeeds.

So we have a predicament: we put dispatch before cleaning to solve the cleanup rules for subsites. But doing so __breaks__ the dispatch rules for non-subsite routes (henceforth called local routes). The solution:

* First, we dispatch to subsites. If a subsite takes the bait, we're in good shape.
* If no subsites match, we apply cleanPath. At this point we do any 301 redirects that are necessary. If not, we continue dispatching to the local routes.
* If nothing matches, return a 404.

I'm actually quite satisfied with this as the final answer: it does not require any double-dispatch attempts, and it gives subsites full control over what their URLs look like.

## The test suite has begun

Greg Weber has long been pushing me to start a proper test suite for Yesod. With these two bugs, this new test suite has [officially begun](https://github.com/snoyberg/yesod-core/blob/master/Test/CleanPath.hs). As anticipated, the test suite is built on top of [wai-test](http://hackage.haskell.org/package/wai-test). I'm hoping that both libraries will improve as a result of this push.

My goal going forward is to create a test case for each reported issue. I want to start the focus on yesod-core, and then expand into some of the additional libraries, especially yesod-form. So let's try and continue the "Please Break Yesod" campaign.
