One of the major goals for the Yesod 0.9 release is a revamp of the yesod-form package. The main change I want to push through is getting rid of the polymorphism introduced by the IsForm typeclass. This typeclass allows functions to work for both monadic and applicative forms (to be covered in a second). However, this introduces a lot of complexity and creates type errors which are impossible to discern.

Instead of spending time comparing and contrasting the current approach to what we want, I'm going to jump straight into a discussion of the goals of the package and what we can do to achieve them. I'm looking for feedback with this post, *especially* feedback on names: I think improving the naming scheme will go a long way towards making yesod-form more approachable.

## Abstract away the view

Let's say I want to ask a user to enter his/her age. This field will be made up of a few components:

* The label
* The input element
* Any error messages from failed validation

How should this be displayed? There are a few choices that immediately pop into mind: a row in a table, some divs, or perhaps some completely customized form like:

    <p>Hey y'all, my name is <input name="name"> and I am <input name="age"> years old.</p>

But we don't want to have a separate intField function for every possible layout. So instead, we need some intermediate datatype that contains this information (along with some other stuff like field ID, if it's required, and a tooltip). Then every field function can create a view based around this data type, and we can provide a few helper functions to map this datatype to some of the standard displays (like a table row).

In the current library, this is called [FieldInfo](http://hackage.haskell.org/packages/archive/yesod-form/0.1.0.1/doc/html/Yesod-Form-Core.html#t:FieldInfo), which I think is a horrible name. Ideas?

## Basic fields, required and optional

A great number of fields can be summed up by three pieces of information:

* How to render our datatype to a Text.
* How to parse a Text to our datatype (returning validation errors as necessary).
* How to create an HTML input form.

In fact, this is such a common pattern that we have a data type for it (currently FieldProfile, open to renaming again). And to generate actual Form functions, we have two helpers: one for required fields, one for optional fields.

These helpers need a little bit more information though: the label, any forced ID or name attributes, and the tooltip. So we have *another* data type, FormFieldSettings. (At this point, I'm not going to mention any more that I'd like to rename things, just take it for granted.)

One little annoyance in the current design is that we need three functions for each datatype: the FieldProfile version, required form and optional form. We'll come back to this later.

## Monadic or applicative?

Here's the real meat of the discussion. yesod-form originally followed the design principles of formlets. formlets heavily use Applicative. This allows very simple construction of forms from smaller pieces, which automatically track error responses and concatenate the HTML for you. In such a system, type signatures will look something like this:

    intField :: Form xml Int

This works out very nicely. Let's say we have a data type that has two Ints (e.g., data MyType = MyType Int Int). Then we can immediately build up a new form:

    myTypeField :: Form xml MyType
    myTypeField = MyType <$> intField <*> intField
    

And this seems like a great system. But eventually, I started getting feature requests for custom forms (you know, the kind of thing that does fit nicely in a table). The Applicative approach presents a bit of a problem, since it doesn't allow direct access to the view code. No problem, we'll just rewrite it as:

    intField :: Form (Int, xml)

Except now we've got another problem: the error handling is still embedded in the Form type. So let's say that the user provides an invalid Int; now this function will not return. This gives us two problems:

* The xml will not be generated for this field at all.
* All subsequent form fields will not be generated.

So we really need something that looks more like:

    intField :: Form (FormResult Int, xml)

Using this system, our code above becomes much more verbose:

    myTypeField = do
        (i1, x1) <- intField
        (i2, x2) <- intField
        return (MyType <$> i1 <*> i2, x1 `mappend` x2)

The originally IsForm-polymorphic approach of yesod-form tried to get around this verbosity by allowing the same intField to simultaneously work as an Applicative and Monadic form. But as I said, we're getting rid of that.

### Possible solution

So here's the system I've been playing around with: we need to except the fact that our code will end up with a few extra keystrokes. We'll have a single function for each datatype, which will give the field profile itself. We'll then have four helper functions:

* areq (applicative required)
* aopt (applicative optional)
* mreq (monadic required)
* mopt (monadic optional)

Some theoretical type signatures would be:

    type AForm xml a -- applicative
    type Form a -- monadic
    
    areq :: FieldProfile xml a -> AForm [xml] a -- we'll explain the list below
    aopt :: FieldProfile xml a -> AForm [xml] (Maybe a)
    mreq :: FieldProfile xml a -> Form (FormResult a, xml)
    mopt :: FieldProfile xml a -> Form (FormResult (Maybe a), xml)

Using the current nomenclature, our built-in functions will all use FieldInfo as their XML type. The Applicative versions want to be able to automatically append XML values together, so we wrap up the FieldInfo in a list.

Finally, we'll want to be able to display our Applicative forms, so we'll have some built-in functions that will convert a list of [FieldInfo] into a Widget. This will simultaneously convert back to a monadic Form data type so that we only need one set of run functions. So we end up with something like:

    formTable, formDivs :: AForm [FieldInfo] a -> Form (FormResult a, Widget)
    runFormGet, runFormPost :: Form x -> Handler (x, Enctype)

Obviously we're glossing over some details like inner monads, site arguments and FormFieldSettings, but I think the overall design should be clear. So now we get pretty close to our original, concise code:

    myTypeForm = runFormPost $ formTable $ MyType
        <$> areq intField
        <*> areq intField

## CSRF protection and missing input

One nifty feature in yesod-form is automatic Cross Site Request Forgery protection. This is a system where every user session has a nonce value associated with it, and no POST form submissions are allowed without that nonce. This prevents someone from creating a nefarious POST form on their site that points to your site.

Totally different issue: some fields like checkboxes need to know whether or not a form was submitted at all. If the form was submitted, and no value is present for this field, it means false. If the form wasn't submitted at all, it means "use the default value." This is a case not handled well by yesod-form right now, and we intend to add support for it. This is easy enough for POST forms: just check the request method. But for GET forms, it's more complicated. The solution: every GET form includes an extra hidden field that indicates that it has been submitted.

If you look closely, both of these use cases involve inserting a little bit of extra HTML into each form. So we can make some minor modifications to the type signatures above to accomodate this:

    formTable, formDivs :: AForm [FieldInfo] a -> Html -> Form (FormResult a, Widget)
    runFormGet, runFormPost :: (Html -> Form x) -> Handler (x, Enctype)

## Devil's in the details

As usual, there are probably a whole bunch of corner cases that have not been addressed here. But I'm hoping this post can spark some design discussions and we can get a strong, stable form package out in the next few weeks.

As far as planning: I'm planning on releasing the new yesod-form to work with Yesod 0.8 so it can be more easily tested. But it will only become the preferred version with the Yesod 0.9 release.
