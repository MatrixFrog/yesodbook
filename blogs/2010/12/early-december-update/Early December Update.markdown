It's been a while since I've given an update on what's going on in the Yesod world. There have been a few minor releases of packages, but most of the work has been in the planning realm. I'm contemplating a number of semi-major changes to infrustructure, at many different levels. I'll put down my thoughts here to hopefully spark some discussion. Don't be surprised to see some of these issues raised on the [web-devel list](http://www.haskell.org/mailman/listinfo/web-devel) or the [web development special interest group](http://www.haskellers.com/teams/4/).

## But first, a word from our sponsors: the founders page

I'm planning on putting up a page on this site for the "founders" of Yesod: a link and a little bio for each of the contributors to Yesod. If you've written some code that's made it into Yesod, send some info over. This includes code in yesod itself, packages like Hamlet, Persistent or WAI, or packages that were written specifically for use with these. In the email, please make sure to mention what you contributed (I don't want to accidently leave out things). I'm sorry for not tracking everyone down personally, I just haven't had a chance to get around to it yet.

And while we're talking about this site, I'm still interested in doing a design refresh. If anyone is interested in contributing some skills for this, please let me know. If someone wants to go all the way and contribute actual patches to the repo, that's great, but I'll gladly accept other forms of input.

## blaze-builder-enumerator and WAI

<div style="background:#eee;padding:0.5em;margin-left:1em"><p><b>Note: the next few paragraphs contain some back information you don't need to know to understand the WAI proposal. Feel free to skip.</b></p>

My new job consists of a lot of XML parsing. If anyone remembers, a few weeks ago I rewrote the Yesod book in a customized XML format, and used John Millikan's libxml-enumerator library. The library is great, and I started using it at work as well. However, some problems cropped up:

* It's a major pain getting libxml set up correctly on a Windows system. John has also written expat-enumerator, which is easier to get installed, but it does not support DOCTYPEs, which is something I needed support for. It also doesn't support XML namespaces properly.

* One tool I needed to write involved parsing an XML file, modifying some of the tags, and spitting it out again. There was no library for rendering the Event datatype.

I solved the first problem by writing an XML parser based on attoparsec. (In case you're wondering: it assumes UTF-8, but comes with an enumeratee to convert UTF-16 and UTF-32 into UTF-8 automatically. Both <abbr title="Big Endian">BE</abbr> and <abbr title="Little Endian">LE</abbr>.) For the latter, I started writing a library, but realized there was no efficient way to produce output from an enumerator. In pure code, I would use blaze-builder, but doing so from an enumerator would involve pulling the entire XML file into memory, which defeats the point of a streaming interface.

I started working on a library to bridge these two wonderful packages, and soon found out that there were already two other efforts going on for this. I passed my code off to Simon Meier, and he is taking over maintainership of <a href="https://github.com/meiersi/blaze-builder-enumerator">blaze-builder-enumerator</a>. (Just to close off the story about XML: once Simon releases blaze-builder-enumerator, I will hopefully be releasing my renderer, parser and some parser combinators I developed for the Yesod book as a package called [xml-enumerator](https://github.com/snoyberg/xml-enumerator).

</div>

That was a very roundabout way of getting to the point at hand: how should we represent request and response bodies in the WAI? I'm currently using a custom-defined enumerator for the response, and something called a source for the request. Bascially, a source allows the input to be paused and resumed, which an enumerator does not. I've considered in the past moving both of them over to an externally defined enumerator datatype. However, WAI currently has incredibly minimalistic dependencies, and I did not have a compelling reason to make the move.

However, blaze-builder-enumerator changes that. It will allow much more efficient implementation of WAI middlewares. For example, the JSONP middleware currently produces some incredibly small bytestrings and sticks them on the beginning and end of the response returned by the application. Ideally, we want to have all our bytestrings come to about 16k, so JSONP is really suboptimal. If instead, our response body became <code>Enumerator Builder IO a</code>, everything would work beautifully again.

The case for moving the request body over to enumerator is less powerful: the current approach is strictly more powerful. However, if we move it over as well, we make the interface simpler, and we get to reuse our tools better.

## A new syntax for Hamlet

Greg Weber has suggested a new syntax for Hamlet templates, and I think the idea has a lot of merit. Instead of reposting his ideas here, I will ask everyone to [read his proposal](http://www.haskell.org/pipermail/web-devel/2010/000569.html). I want to hear everyone's opinion, whether it is yay, nay, or don't care. And I don't care if this turns into a bikeshedding contest: we want to do this right. It will be much more difficult to make a change like this after Yesod 1.0.

## mime-mail + enumerator, ASCII and Text

I recently released mime-mail 0.1 to add support for quoted-printables, encoded headers and headers on individiual parts. I still have a few open API questions that I'd like feedback on:

* The parts of a message are given as lazy bytestrings. This means you have to use lazy IO for attaching files. Another approach would be to use an enumerator here (even using blaze-builder-enumerator again possibly). However, this might add too much complexity to the API. I could also create some kind of <code>data PartBody = PartLBS ByteString | PartEnumerator ... | PartFile FilePath</code>, which might be a good trade-off between power and simplicity.

* While header values can be any Unicode value, header names are required by the spec to be ASCII-only. mime-mail currently encodes them as UTF-8, which is definitely the *wrong* thing to do. I can think of a few possibilities of right things:<ol><li>Throw a runtime exception when a non-ASCII value is given.</li><li>Change the datatype to bytestring and let the user worry about it.</li><li>Create an Ascii datatype that only allows valid ASCII characters in.</li></ol>

* I'm still using Strings for the header values. Is it time to move over to Text for this? Internally, they get converted to Text anyway.

## Forms

Forms in Yesod are becoming a major codebase all on their own. I've been considering moving it out into a separate package to make it easier to make breaking changes to the API. However, I just heard about Jasper's [digestive-functors](http://hackage.haskell.org/package/digestive-functors) package, which believe it or not is about forms and not laxatives (I kid, I kid). It looks like a better version of formlet, supporting some features such as inline error messages.

If this library offers what we need in Yesod, I would like to try to move to it. I believe that whenever possible it's great to use existing tools. I have not yet had a chance to look into this library completely, so I don't know if it has all the features we need. But if I do move Yesod's forms into a separate package, then it could be a user choice which form library to use.

## Yesod 0.7

The magnitude of some of these changes really necessitates a 0.7 release: I don't feel comfortable making such sweeping modifications into the 1.0 release. This will also be a time to update any underlying libraries that have become out-of-date, such as neither. If anyone has any other kinds of thing they would like to see changed at the same time, let me know. I do not have a timetable for 0.7, since it really depends on how much of the above we decide to do.
