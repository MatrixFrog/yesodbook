I'm going to describe some projects that are on my wishlist: given an infinite amount of time and an infinite number of snacks, I would write all of these. However, I *do* live in the real world, so I don't have time to write and maintain everything I'd like. I'm going to post my ideas here to

* not lose them,

* try to see which of these the community would be most interested in, and

* see if anyone out there is interested in writing one of these him/herself.

If anyone wants to take a crack at one of these, let me know (either comments or email). I'd be more than happy to provide any guidance and support people want. I'll try to rate these projects with a difficulty level: 1 being you don't know monads yet, 10 being your name is Simon.

__Warning: difficulty levels are made up on the spot and may be completely mistaken. Do not blame me if you stay awake three weeks on end trying to implement what I guessed would be an easy task.__

## wai-vhost

I emailed the web-devel list about this the other day. It's a very basic idea: provide a WAI middleware which routes requests to one of many different applications based on the Host request header, just like a webserver does. You can [read my original email](http://www.haskell.org/pipermail/web-devel/2010/000541.html).

__Difficulty level: 2.__

## web-extras

This could easily be split up into a bunch of different packages. The idea is to provide quality support for services like Gravatar, Disqus, Google Analytics, Recaptcha, and more. None of these are very complicated, it's just finding a balance between nice API and providing flexibility. A lot of this code already exists in the Haskellers website and just needs to be factored out.

__Difficulty level: 3.__

## yesod-markdown

I don't natively provide support for Pandoc in Yesod because 1) it's a large dependency and 2) it's GPLed. However, it's often desireable to use Markdown syntax in a website, and it would be great to have that packaged up somewhere. This package would provide some Markdown datatype, form functions, automatic rendering to HTML, serialization to Persistent, etc.

__Difficulty level: 3.__

## wai-unit

I want to start testing Yesod thoroughly, but I haven't written the test framework yet. This doesn't necessarily need to be a WAI-specific test framework: it would be even more useful if it used http-enumerator and spoke properly over a socket. But starting as WAI-only would probably be easier, and having a WAI-adapter would be incredibly convenient. The trick here is getting a nice, thorough, powerful API.

__Difficulty level: 5.__

## attoparsec-text

I'm relying more and more on the text package, but unfortunately there isn't (to my knowledge) a nice parser combination library for it. It would be wonderful if someone wrote one with a goal towards the same performance characteristics of attoparsec. Almost certainly, a text-based one will be slower, but it would be a nice goal nonetheless.

If that gets written, the next thing I would want is an adapter for the enumerator package, but that would likely be fairly easy.

__Difficulty level: 7.__

## Native XML parser with enumerator interface

I'm using libxml-enumerator for the Yesod book these days, and like it a lot. I've put together some basic combinators to ease the job of building up my data value, and will hopefully get those into a package soon. I liked this library so much, I tried to use it for work. One problem: I can't for the life of me get libxml working on Windows properly. (Any advice on getting Windows set up properly to build with pkg-config is welcome.)

I've worked around it with expat-enumerator, but it seems that namespace handling in that library is not as powerful. It's not a problem for me at the moment, but could be in the future. A native-Haskell parser (hint: built on attoparsec-text) would be very useful here. Also, it would be amazing to get line-number information in there too.

Oh, and I also think that a rendering library would be useful, but that's in the works already.

__Difficulty level: 6.__
