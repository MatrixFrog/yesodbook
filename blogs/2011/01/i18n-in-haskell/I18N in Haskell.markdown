I got a feature request from Антон Чешков to allow the messages in the yesod-auth package to be user-customizable. Adding such a feature is fairly simple: I added a whole bunch of methods to the YesodAuth typeclass, one for each method. While that works, I'm not happy with the result. Besides being an ugly solution, there is a much more fundamental problem: *it only support one language*.

Yesod does in theory have i18n support. There is a function for getting a list of languages that the user accepts, and support for using cookies to override the Accept-Language field. This is all well and good, but it's like saying a car supports flying, all you need is a catapult. (Sorry, I couldn't resist a car analogy.)

In theory, I think multi-lingual support needs to be added at the *template* level, in Yesod's case, Hamlet. However, we first need to figure out the right approach to I18N for Haskell in general. Consider what I'm about to present a straw man: I want to hear why this is such a bad idea.

Let's start by making a bold assertion: gettext is a huge, ugly hack. Sure, it works fine for static strings like "Hello World". Of course, there's no compile-time checks to make sure you didn't accidently use "Hello world" (lower-case w) or "Hello World!". But that's not even the big hack: imagine I want to output something as simple a "\<count\> \<object\>(s)". gettext has trouble with this because:

* We need to be able to pluralize things.
* Believe it or not, not every language puts their numbers before their objects. (In Hebrew, for instance, you can say the equivalent of "house one" or "two houses".)

I'm sure others with more gettext experience than I can explain both how to work around these issues, and how there are even *worse* hacks involved. Frankly, that's not our issue here. Let's see how we can leverage the strengths of Haskell for this. I present my strawman (explanation below):

<script src="https://gist.github.com/778712.js?file=i18n.hs"></script>

I start off with the MaybeRead typeclass. It's not really i18n specific, it's just used for parsing the language datatype from strings. The meat of this sample is lines 8-12. We define an I18N typeclass with two associated datatypes: a datatype to represent languages, and an output datatype for displaying the messages.

So why bother with the language datatype? After all, there *are* international language codes that we could rely on. The reason is twofold: we can parse our language code *once* into a Haskell datatype, helping performance, and more importantly, we can pattern match and the compiler will tell us if we missed a translation. I think the message datatype is more obvious: some people will want String, some Text, and some Html.

i18n tries to translate your datatype into a message. It returns its value in a Maybe to let you know that there is no translation for the specified language available. There is also i18nDefault, which gives the default translation for a value.

Since we can limit the range of languages using the Language associated type, why not just have i18n return the value outside of Maybe and skip the i18nDefault? A few reasons:

* It's likely that people will want to use more generic Language datatypes, which would include languages that their application does not support.

* In the context of web applications, you don't just get a requested language, you get a prioritized list of languages. This allows you to try out different languages until you get a match.

The two example helper functions, i18nLang and i18nLangs, demonstrate two standard use cases for this class: translated to a specific language, and translated to one of many languages. The rest of the gist is a simple example of translated messages for a number guessing game.

Oh, and since the code to generate the messages is actual Haskell, you can due any kinds of manipulations you want. You are *not* constrained to the order in which the arguments are provided, you can use any logic you want to make plural words, you can even do complex date formatting or "fuzzy times" if you want.

Anyway, it's almost 1 in the morning here, so perhaps this won't be too coherent, but I hope we can come up with a good general solution for i18n in Haskell. Let me know what you think!
