I mentioned about two weeks ago on the web-devel list that I wanted to have a centralized site for Haskellers to put up professional profiles and for employers to not only find candidates, but also have confidence that Haskell is a language with a strong community of talented people.

I asked for feedback, and got a lot of it. I think the thing most demanded by the community was OpenID logins. I'm happy to say that [we now have that](http://www.haskellers.com/), and frankly not much else. The reason is that I've spent the past two weeks working towards the goal of OpenID logins; the rest of this post is the explanation of why. Now that I've finally gotten that problem tackled, I can start implementing the cool features we want in Haskellers (you know, like a website).

## authenticate package, 2 weeks ago

You might be surprised that I'm complaining about OpenID support; after all, I wrote the [authenticate](http://hackage.haskell.org/package/authenticate) package, which has OpenID support built in. The problem is that it only supported OpenID version 1, and most of the big OpenID providers (Google, Yahoo!) have moved on to version 2.

I've been meaning to add support for OpenID 2 for a while, but I've kind of had an easy out with RPXNow; they are a proprietary site that makes it easy to implement 3rd-party login with a number of different providers. However, I didn't feel like going the proprietary route at all on Haskellers, and decided to bite the bullet and get OpenID 2 support done right.

## openid package

Trevor Elliott put together an [openid](http://hackage.haskell.org/package/openid) package which would seem to complement authenticate perfectly: it supports OpenID 2 but not 1. Unfortunately, I was never able to get it to work right out of the box. It seemed to have some issues with HTTPS support; if I'm not mistaken, there was an API change in the HTTP package that caused openid's HTTPS support to fail.

This brought up another annoyance I've had: a lack of a good HTTP library in Haskell. So I decided that while I was on this crusade of providing proper OpenID 2 support, I may as well pick up a few quests as well. The result is [http-enumerator](http://hackage.haskell.org/package/http-enumerator). I'm quite happy with it; some of its main features are:

* An incredibly straight-forward API.

* Built in HTTPS support using either OpenSSL or Vincent Hanquez's [tls package](http://hackage.haskell.org/package/tls).

* By building on top of the [enumerator](http://hackage.haskell.org/package/enumerator) package, you can do nice things like constant-memory download to a file. Working on this also inspired me to write [two](http://docs.yesodweb.com/blog/enumerators-tutorial-part-1/) [tutorials](http://docs.yesodweb.com/blog/enumerators-tutorial-part-2/) on the enumerator package (third one is in the works).

I was then able to gut out the HTTP library dependency in the openid package and replace it with http-enumerator. Using this, I could now log in to Yahoo!, but Google support did not work.

## Associations

It turns out the Google issue had to do with associations. For those not familiar: there are two ways you can check if a user really logged in with an OpenID provider: set up an association in advance which uses some cryptographic techniques to allow you to check a signature, or simply ask the provider after the user comes back to your site if the login was real. Associations support ended up causing three problems for me:

* For some reason it wasn't working with Google, as mentioned above.

* It introduced a dependency on the OpenSSL library. I'm trying to make Yesod have as few system dependencies as possible.

* It depends on the [AES package](http://hackage.haskell.org/package/AES), which has an outstanding issue with 32 bit systems. This is problematic, since many VPS servers install 32 bit OSes to conserve memory.

## Frankenstein

OpenID 2 is a complicated protocol, and Trevor got all of the complicated stuff (XRDS, for example) handled properly. So instead of reinventing the wheel, I decided to simply pull the necessary code directly into the authenticate package and remove the associations support. At first, this resulted in an extraneous Web.Authenticate.OpenId2 module, which seemed a bit strange.

After some more massaging, I was able to combine the OpenID 1 and 2 support into a single interface. This means file are downloaded once and checked for both OpenID 1 and 2 support, and everything happens transparently for the user. I've released this as authenticate 0.6.6, and the cool thing is **it has the exact same API as before**, so if you've been using this package, you just got OpenID 2 support for free.

## yesod-auth 0.1.0

I also just uploaded a new version of the [yesod-auth](http://hackage.haskell.org/package/yesod-auth). For those not aware: the main yesod package, through version 0.5.*, includes a Yesod.Helpers.Auth module. Starting from the next major release of Yesod, this will be removed, in favor of having this functionality in a separate package.

This release just offers some minor enhancements to the API, such as replacing defaultRoute with loginRoute and logoutRoute. It's worth checking out this package if you're using Yesod, especially if you need to implement a custom authentication scheme.

## Back to Haskellers

Anyway, that officially clears my immediately pending coding queue, so I can start working properly on Haskellers tomorrow. Right now, it's just a basic site where you log in and enter a few fields of information. I'm planning on adding:

* A skills section users can fill out.

* Email address validation.

* Recaptcha protection for account creation and viewing user email addresses.

* A verified user feature, meaning some human has verified you are who you claim to be.

* A full RESTful, JSON-based API for querying data from the site. I'm hoping that we, as a community, can start trying to create more interoperative services like this.

There'll be some more stuff as well, but if you have any thoughts on what you'd like to see in the site, please let me know!
