It's been a busy week for the Yesod team. The big API changes have been going into the Forms package, which I'll describe shortly. But first, we wanted to let everyone know about some of the other excitement.

* Greg has implemented [html5boilerplate](http://html5boilerplate.com/) integration for the scaffolded site.
* Mark Wotton has set us up on his [continuous integration](http://ci.shimweasel.com/job/yesod/) server. Greg is continuing the process of switching our test suites over to the new "cabal test" system so they can be run automatically on the CI server.
* Erik de Castro Lopo has implemented a slew of enhancements for http-enumerator, focusing on proxy support and HTTP basic authentication. We're still adding a few extra features, so the code is not yet on Hackage.
* I've put in a lot of work on a new documentation system to replace the current website. I hope to blog more about this soon, but I'm not sure how much I'm allowed to say at the moment. I *will* say that one of the major features will be first-class user-contributed documentation, directly from the site.
* We've got a new [contributors page](/contributors). We're starting to get the Yesod team a bit more organized and try to make it clear to the otuside world who is working on which pieces of the puzzle; expect such information to appear on that page in the future.
* We're starting a [testimonials page](/testimonials). There are quite a number of individuals and companies using Yesod in production now. If you're one of them, and you'd like to tell the world about it (and get a bit of free advertising in the process), [send me an email](mailto:michael@snoyman.com).

## html5boilerplate

Perhaps the biggest pain for web developers since the second web browser was created is cross-browser compatibility. The job of a framework is to deal with the common pain points like these. Using browser-independent javascript libraries is a huge win here, and the default Yesod widgets use jQuery and jQuery UI. But there are still html and css issues waiting to trip you up, including when trying to leverage newer html5 features and maintain browser compatibility. Html5boilerplate can help us out here. It is definitely a wide-ranging project- it tries to cover a lot of issues in a backend programming language agnostic way. For many aspects this is not optimal and we are already handling them appropriately within Yesod. But we can leverage their html layout and css file to provide proven techniques for dealing with cross-browser issues.

We want to keep things simple and avoid intimidating the first-time user. So the new scaffolding tool silently generates these files:

* hamlet/boilerplate-layout.hamlet
* static/css/html5boilerplate.css

The file: hamlet/default-layout.hamlet is still the default. But when you are serious about launching, you will want to switch to use html5boilerplate. Or if you have your own tried and true techniques, just delete these files.

## Forms
__Note__: Just to make it clear, these "decisions" are still up for discussion. Greg has some ideas on how to clean up the API a bit, and I'd rather delay the release if it means a better final product.

I [blogged recently about the Yesod forms library](http://www.yesodweb.com/blog/yesod-form-revamp), leaving some stuff up in the air. Well, as part of the work on the i18n efforts for Yesod in general, and work on the new documentation site, I've hammered out a number of the details. There are still a few changes to be made, and I haven't decided if it will be released before the 0.9 release. But it's mostly complete. This post will give a high-level overview.

## Three types of forms

There are three types of forms:

* Monad (Form), which does not handle error propogation and view (i.e., the HTML itself). It's very convenient for making custom forms.

* Applicative (AForm), which handles error propogation and view automatically, allowing for very concise code. If you need to create a standard form, such as a table layout form, this is what you'd use.

* Input (FormInput). This is a special type for simply reading GET/POST parameters without actually constructing any HTML. In the current form package, there is a whole class of stringInput, intInput, etc functions. However, they all lived in the standard form datatype; having a separate type is new.

You can run monadic forms with runFormPost/runFormGet, and input forms with runInputPost/runInputGet. For Applicative forms, you must first convert them to monadic forms, using a render function. The package currently has renderTable and renderDivs.

## Field

An individual field needs three pieces of information: how to parse from textual data, how to render to textual data, and how to create a view (widget). That's where the Field datatype comes into play. It looks like:

    data Field xml msg a = ...

The xml type argument specifies the datatype for the view; this will usually be a Widget. And <code>a</code> specifies the datatype the field will return (e.g., intField returns an Int). But the msg is the interesting one: yesod-forms is now fully i18n'ed! Instead of just returning a message like "Invalid integer", intField returns an InvalidInteger value of type FormMessage.

yesod-core now includes a typeclass:

    class RenderMessage master message where
        renderMessage :: master -> Languages -> message -> Text

In order to use the forms library, you're going to need to provide an appropriate RenderMessage instance. If you just want to use the default rendering (which always displays English), it would look like:

    instance RenderMessage MyApp Form where
        renderMessage _ _ = defaultFormMessage

So this is a bit of overhead in order to use the library, but the advantages of being able to easily internationalize an application (IMO) are well worth it.

## Fields to Forms

We have six functions for turning a Field into one of the Form types above. We have a required and optional variant, and a different function for each type of form. The naming is very straight-forward: mreq, mopt, areq, aopt, ireq, and iopt. The first four take three arguments: the Field itself, a FieldSettings (we'll get there) and the initial value for the field. The last two only take two arguments: the Field, and the parameter name.

So what's this FieldSettings? It's a data type containing four pieces of information: the label of the field, the tooltip, the ID and the name. Only the first is required. Oh, and here's the important bit: the label and tooltip can be any datatype, allowing for internationalized fields again! So for example:

    data MyMessage = Name | Age

    myForm = runFormPost $ renderTable $ (,)
        <$> areq textField (FieldSettings Name Nothing Nothing Nothing) (Just "Michael")
        <*> aopt intField (FieldSettings Age Nothing Nothing Nothing) Nothing

If you're not using i18n, you can go ahead and use plain old Text. And in fact, we even have a convenient IsString instance, so if you turn on OverloadedStrings, you could write the above as:

    myForm = runFormPost $ renderTable $ (,)
        <$> areq textField "Name" (Just "Michael")
        <*> aopt intField "Age" Nothing

## FormResult

A form can return three possible results:

1. There was no data present
2. There was invalid data present
3. There was good data present

We use FormResult for this, completely unchanged from previous versions:

    data FormResult a = FormMissing | FormFailure [Text] | FormSuccess a

## FieldView

You usually won't need to interact directly with a FieldView. This datatype contains information on how a field needs to be rendered, such as the label, HTML for input, and any error messages. renderTables/renderDivs use it internally; if you decide to write a custom rendering function, you'll need it too.
