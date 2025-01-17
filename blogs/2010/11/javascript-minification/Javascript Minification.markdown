We all know that serving the fastest web page possible is the "in" thing these days. We await with bated breath every piece of advice from Google, Yahoo! and Facebook. (Whatever you think of the policies of any of these companies, we have to admit they *do* handle insane volumes of traffic.)

One of these techniques is to shrink down our responses as much as possible. Yesod already helps out on this front: Hamlet and Cassius both produce very terse code, and the default scaffolded Yesod site concatenates together your CSS and Javascript code into single files that can be served with caching.

But the one piece of the puzzle that has been missing up until now has been minifying our Javascript. I put out a request for someone to address this, and Alan Zimmerman took up the challenge. The result is [hjsmin](http://hackage.haskell.org/package/hjsmin). It is a **very** well tested piece of software, and seems by all accounts to be perfectly safe to use.

I wanted to plug it straight in to [Haskellers.com](http://www.haskellers.com/), but soon realized there were some performance issues. Minifying the typical Javascript response of an individual page took about 150ms on my local system. That may not seem like much, but it will effectively kill any performance gains of using minification.

With that in mind, Alan and I started working on optimizing hjsmin. I have no doubt that the performance will be seeing some great strides in the future, but I'm impatient: I want to minify **now**!

## Caching to the rescue

The solution is actually rather simple, and in fact is something I should have done a while ago. Let's review how Haskellers.com serves Javascript (CSS is the same):

1) Run through all the handlers, which produce a bunch of Javascript code.

2) Concatenate all of that code together into a lazy bytestring. (By the way, under the surface we're using blaze-builder for even better performance.)

3) Get an MD5 hash of the content.

4) Store the Javascript in a file with the hashed name.

5) Insert a &lt;script&gt; tag into the resulting HTML page referencing that Javascript.

Forget about minification for a second. Step four is pretty stupid: if a million people request my homepage, the same Javascript will be generated a million times, __and the same file will be written a million times__. What was I thinking? Let's replace step four with:

1) Check if a file exists with the hashed named.

2) If not, write the content to the file.

And suddenly instead of a million file writes, one file write and a million file existence checks (which are **much** cheaper). Note that this optimization works equally well for CSS. But also notice that writing content now only happens once. I'm going to modify the above process just a little bit more.

1) Check if a file exists with the hashed named.

2) If not, write the **minified** content to the file.

And voila! The expensive minification only happens once. The trick to remember is the filename hash is calculated **based on the non-minified Javascript**. This code is now running live on Haskellers. You can [see the relevant code on Github](https://github.com/snoyberg/haskellers/commit/2401342e488caae86fdb87f5280656473fe0af13).

Please let me know if you see any misbehaving Javascript, but I'm not too worried. Assuming all goes well, expect to see this code as part of the new Yesod site template.
