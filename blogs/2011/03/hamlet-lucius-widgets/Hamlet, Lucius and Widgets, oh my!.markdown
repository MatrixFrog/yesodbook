I've just released version 0.7.3 of the hamlet package, which introduces two noteworthy changes. The first is improved conditional class support. For those who are not familiar, Hamlet allowed you to put in conditional attributes like so:

    <option :isSelected:selected=true>My Option

In this case, if the value of isSelected is True, then the option tag will have a selected attribute with a value of true. option and input[type=checkbox] tags were the original use case for this feature. But another common use is with classes, e.g.:

    <a href=@SomeRouteR@ :isCurrentRoute SomeRouteR:class=current>

The problem with this is that it doesn't play nicely with the .class syntax Hamlet provides: using both would mean that you end up with two class attributes, which means one will be ignored by the web browser. The correct approach is to space-separate your classes. Hamlet 0.7.3 makes two changes to help you out here:

* If you have a class=someclass and a .otherclass, Hamlet will automatically combine them into class="someclass otherclass".
* You can now put conditionals on .classes, eg: <a href=@SomeRouteR@ :isCurrentRoute SomeRouteR:.current

Thanks to Blake Rain for bringing this up.

The other new feature in this release is Lucius: a CSS template language. This is meant as a sister language to Cassius. While Cassius uses whitespace for delineation, Lucius uses braces and semicolons. This means that you can insert CSS verbatim into Lucius and it will work.

There are still lots of features to be added: @media and @import don't work yet, for instance. And then there's some extra features we would like to implement from Less, like nested blocks. For the moment, Lucius and Cassius cannot be mixed within the same template, though they use the same datatypes and can therefore be passed to the same functions. Is there desire in the community to allow them to live in the same templates?

And finally, we come to widgets. Sven Koschnicke [put up a very good example](http://www.haskell.org/pipermail/web-devel/2011/001055.html) of a horrible performance issue in Yesod. This was a great example of how to properly debug a performance bug. Or rather, it was a reminder to me that I should never assume I know where the bug was coming from. My testing basically came down to:

* Oh, he's doing lots of DB code. That must be the problem. (Do some profiling...) Oh, that's not the problem.
* Oh, replacing addWidget with addHtml speeds things up dramatically. Must be a polymorphic Hamlet bug caused by the inliner. (More profiling...) Oh, wrong again.
* Huh, I forgot that I programmed the implementation of Widget when I was drunk.

Turns out the real problem was the 8-level monad transformer stack powering widgets. Instead of that, I simply replaced it all with a RWS (read-write-state) transformer with a single datatype containing all the data previously in the stack. As is usually the case, I think I went for the transformer stack approach due to some misguided ideas on performance. As usual, if you want to know about the performance of something, you have to benchmark. So here are the results of the Widget bigtable benchmark:

<table border="1" cellpadding="5">
<caption>Criterion BigTable results, microseconds (lower is better)</caption>
<tr><th>Version</th><th>Time</th></tr>
<tr><td>Original code</td><td>23,700</td></tr>
<tr><td>Lazy RWS, non-strict datatype</td><td>141</td></tr>
<tr><td>Strict RWS, non-strict datatype</td><td>298</td></tr>
<tr><td>Lazy RWS, strict datatype</td><td>101</td></tr>
<tr><td>Strict RWS, strict datatype</td><td>102</td></tr>
</table>

Yes, those numbers are real. All Yesod users should probably send Sven a beer for helping discover this bug. I've released version 0.7.0.2 of yesod-core to change the definition of Widget appropriately, using the lazy RWS transformer and a strict datatype.
