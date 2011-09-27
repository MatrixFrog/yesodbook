By Greg Weber

<img src="/static/warpspeedahead.png">

The Yesod team is proud to announce a more stable, performant 0.7.1
release. The new Warp web server was as fast as promised, but suffered
from a timeout deadlock bug. Not only has that been squashed, but the fastest Haskell web server
just got even faster: Matt Brown's improvements made Warp 20% faster on the
pong benchmark!

<img src="/static/benchmarks/warp0321.png">

Other minor bug fixes have been committed to Yesod, so we believe
the release is very solid now. Please upgrade! A new version of hint for GHC7 has just been
released, so go ahead and develop on 7 if you want. We are waiting for
the 7.0.2 release (in a few weeks hopefully) before recommending using GHC7 in production.

One of the most visible 0.7 changes was with hamlet syntax. We hope that this new
syntax will

1) lower the barrier to entry
2) appeal to a wider variety of programmers and
3) appeal to designers.

<code><pre>
    &lt;table#an-id.a-class style=display:none;>
     &lt;thead>
       &lt;tr>
         &lt;th>fruit
         &lt;th>color
     &lt;tbody>
       &lt;tr>
         &lt;td>
           &lt;a href=@{FruitR fid}>#{fruitName f}
         &lt;td>#{fruitColor f}
</pre></code>

There is no need to learn a radical syntax here.
Just by making white space significant and removing closing tags, we eliminate the main source of invalid html.
With that, and some id/class shortcuts, and making attribute quoting optional, we have eliminated the main sources of tedium in html.

We think Yesod has a lot of great code, and we want to share it with the
community. Yesod 0.7 is more modular than ever- it is now broken up
into several main packages (yesod-core, yesod-form, yesod-auth, yesod-static, etc),
and there are new packages being split out (cookie and pool in this release).
 We aren't just throwing things over the wall here- some of these we are hoping will
become widely used even outside the web development community. json-enumerator is already a fast json
generator, and we are planning on making it a complete json implementation.

Part of the modularity of Yesod is being built on top of
an improved low-level http interface (WAI 0.3). This interface is mostly
about sharing code in the web development community, but we also feel it
is going to become an option to code apps directly against Wai when you
aren't happy with the abstractions Yesod provides or if you are creating
a very simple application.

Yesod has been a fully-featured, highly productive framework for some
time now. But we haven't been content, and we keep striving to make things
better. We know that it needs to be easier for new users-
 and we are now putting a big focus on documentation before making a 1.0 release.

The Haskell web development scene in general is starting to get legs.
We think 2011 will mark the start of serious web development in the Haskell community.
We are excited to be a part of it, and to be able to offer Yesod 0.7.1 to the community today.
