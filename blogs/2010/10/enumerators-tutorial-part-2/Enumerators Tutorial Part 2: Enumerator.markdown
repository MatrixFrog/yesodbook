
This content is now part of the [Yesod book](http://www.yesodweb.com/book/enumerator). It is recommended to read there, since the content is more up-to-date.

Note: code for this tutorial is available as [a github gist](http://gist.github.com/607856).

## Extracting a value

When we finished the [last part of our tutorial](http://docs.yesodweb.com/blog/enumerators-tutorial-part-1/), we had written a few iteratees, but we still didn't know how to extract values from them. To start, let's remember that Iteratee is just a newtype wrapper around Step:

    newtype Iteratee a m b = Iteratee { runIteratee :: m (Step a m b) }

First we need to unwrap the Iteratee and deal with the Step value inside. Remember also that Step has three constructors: Continue, Yield and Error. We'll handle the Error constructor by returning our result in an Either. Yield already provides the data we're looking for.

The tricky case is Continue: here, we have an iteratee that is still expecting more data. This is where the EOF constructor comes in handy: it's our little way to tell the iteratee to finish what it's doing and get on with things. If you remember from the last part, I said a well-behaving iteratee will never return a Continue after receiving an EOF; now we'll see why:

    extract :: Monad m => Iteratee a m b -> m (Either SomeException b)
    extract (Iteratee mstep) = do
        step <- mstep
        case step of
            Continue k -> do
                let Iteratee mstep' = k EOF
                step' <- mstep'
                case step' of
                    Continue _ -> error "Misbehaving iteratee"
                    Yield b _ -> return $ Right b
                    Error e -> return $ Left e
            Yield b _ -> return $ Right b
            Error e -> return $ Left e

Fortunately, you don't need to redefine this yourself: enumerator includes both a [run](http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:run) and [run\_](http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:run_) function. Let's go ahead and use it on our sum6 function:

    main = run_ sum6 >>= print

If you run this, the result will be 0. This emphasizes an important point: an iteratee is not just *how* to process incoming data, **it is the state of the processing**. In this case, we haven't done anything to change the initial state of sum6, so we still have the initial value of 0.

To give an analogy: think of an iteratee as a machine. When you feed it data, you modify the internal state but you can't see any of those changes on the outside. When you are done feeding the data, you press a button and it spits out the result. If you don't feed in any data, your result is the initial state.

## Adding data

Let's say that we actually want to sum some numbers. For example, the numbers 1 to 10. We need some way to feed that into our sum6 iteratee. In order to approach this, we'll once again need to unwrap our Iteratee and deal with the Step value directly.

In our case, we know with certainty that the Step constructor we used is Continue, so it's safe to write our function as:

    sum7 :: Monad m => Iteratee Int m Int
    sum7 = Iteratee $ do
        Continue k <- runIteratee sum6
        runIteratee $ k $ Chunks [1..10]

But in general, we won't know what constructor will be lying in wait for us. We need to properly deal with Continue, Yield and Error. We've seen what to do with Continue: feed it the data. With Yield and Error, the right action in general is to **do nothing**, since we've already arrived at our final result (either a successful Yield or an Error). So the "proper" way to write the above function is:

    sum8 :: Monad m => Iteratee Int m Int
    sum8 = Iteratee $ do
        step <- runIteratee sum6
        case step of
            Continue k -> runIteratee $ k $ Chunks [1..10]
            _ -> return step

## Enumerator type synonym

What we've done with sum7 and sum8 is perform a transformation on the Iteratee. But we've done this in a very limited way: we've hard-coded in the original Iteratee function (sum6). We could just make this an argument to the function:

    sum9 :: Monad m => Iteratee Int m Int -> Iteratee Int m Int
    sum9 orig = Iteratee $ do
        step <- runIteratee orig
        case step of
            Continue k -> runIteratee $ k $ Chunks [1..10]
            _ -> return step

But since we always just want to unwrap the Iteratee value anyway, it turns out that it's more natural to make the argument of type Step, ie:

    sum10 :: Monad m => Step Int m Int -> Iteratee Int m Int
    sum10 (Continue k) = k $ Chunks [1..10]
    sum10 step = returnI step

This type signature (take a Step, return an Iteratee) turns out to be very common:

    type Enumerator a m b = Step a m b -> Iteratee a m b

Meaning sum10's type signature could also be expressed as:

    sum10 :: Monad m => Enumerator Int m Int

Of course, we need some helper function to connect an Enumerator and an Iteratee:

    applyEnum :: Monad m => Enumerator a m b -> Iteratee a m b -> Iteratee a m b
    applyEnum enum iter = Iteratee $ do
        step <- runIteratee iter
        runIteratee $ enum step

Let me repeat the intuition here: the Enumerator is transforming the Iteratee from its initial state to a new state by feeding it more data. In order to use this function, we could write:

    run_ (applyEnum sum10 sum6) >>= print

This results in 55, exactly as we'd expect. But now we can see one of the benefits of enumerators: we can use multiple data sources. Let's say we have another enumerator:

    sum11 :: Monad m => Enumerator Int m Int
    sum11 (Continue k) = k $ Chunks [11..20]
    sum11 step = returnI step

Then we could simply apply both enumerators:

    run_ (applyEnum sum11 $ applyEnum sum10 sum6) >>= print

And we would get the result 210. (Yes, (1 + 20) * 10 = 210.) But don't worry, you don't need to write this applyEnum function yourself: enumerator provides a [$$](http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:-36--36-) operator which does the same thing. Its type signature is a bit scarier, since it's a generalization of applyEnum, but it works the same, and even makes code more readable:

    run_ (sum11 $$ sum10 $$ sum6) >>= print

<code>&#36;&#36;</code> is a synonym for <a href="http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:-61--61--60--60-">==&lt;&lt;</a>, which is simply <code>flip <a href="http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:-62--62--61--61-">&gt;&gt;==</a></code>. I find <code>&#36;&#36;</code> the most readable, but <abbr title="your mileage my vary">YMMV</abbr>.

## Some built-in enumerators

Of course, writing a whole function just to pass some numbers to our sum function seems a bit tedious. We could easily make the list an argument to the function:

    sum12 :: Monad m => [Int] -> Enumerator Int m Int
    sum12 nums (Continue k) = k $ Chunks nums
    sum12 _ step = returnI step

But now there's not even anything Int-specific in our function. We could easily generalize this to:

    genericSum12 :: Monad m => [a] -> Enumerator a m b
    genericSum12 nums (Continue k) = k $ Chunks nums
    genericSum12 _ step = returnI step

And in fact, enumerator comes built in with the [enumList](http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:enumList) function which does this. enumList also takes an Integer argument to indicate the maximum number of elements to stick in a chunk. For example, we could write:

    run_ (enumList 5 [1..30] $$ sum6) >>= print

(That produces 465 if you're counting.) The first argument to enumList should never affect the result, though it may have some performance impact.

Data.Enumerator includes two other enumerators: [enumEOF](http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:enumEOF) simply passes an EOF to the iteratee. [concatEnums](http://hackage.haskell.org/packages/archive/enumerator/0.4.0.2/doc/html/Data-Enumerator.html#v:concatEnums) is slightly more interesting; it combines multiple enumerators together. For example:

    run_ (concatEnums
            [ enumList 1 [1..10]
            , enumList 1 [11..20]
            , enumList 1 [21..30]
            ] $$ sum6) >>= print

This also produces 465.

## Some non-pure input

Enumerators are much more interesting when they aren't simply dealing with pure values. In the first part of this tutorial, we gave the example of the user entering numbers on the command line:

    getNumber :: IO (Maybe Int)
    getNumber = do
        x <- getLine
        if x == "q"
            then return Nothing
            else return $ Just $ read x

    sum2 :: IO Int
    sum2 = do
        maybeNum <- getNumber
        case maybeNum of
            Nothing -> return 0
            Just num -> do
                rest <- sum2
                return $ num + rest

We referred to this as the pull-model: sum2 pulled each value from getNumber. Let's see if we can rewrite getNumber to be a pusher instead of a pullee.

    getNumberEnum :: MonadIO m => Enumerator Int m b
    getNumberEnum (Continue k) = do
        x <- liftIO getLine
        if x == "q"
            then continue k
            else k (Chunks [read x]) >>== getNumberEnum
    getNumberEnum step = returnI step

First, notice that we check which constructor was passed, and only perform any actions if it was Continue. If it was Continue, we get the line of input from the user. If the line is "q" (our indication to stop feeding in values), we do nothing. You *might* have thought that we should pass an EOF. But if we did that, we'd be preventing other data from being sent to this iteratee. Instead, we simply return the original Step value.

If the line was not "q", we convert it to an Int via read, create a Stream value with the Chunks datatype, and pass it to k. (If we wanted to do things properly, we'd check if x is really an Int and use the Error constructor; I leave that as an exercise to the reader.) At this point, let's look at type signatures:

    k (Chunks [read x]) :: Iteratee Int m b

If we simply left off the rest of the line, our program would typecheck. However, it would only ever read one value from the command line; the <code>&gt;&gt;== getNumberEnum</code> causes our enumerator to loop.

One last thing to note about our function: notice the b in our type signature.

    getNumberEnum :: MonadIO m => Enumerator Int m b

This is saying that our Enumerator can feed <code>Int</code>s to any Iteratee accepting <code>Int</code>s, and it doesn't matter what the final output type will be. This is in general the way enumerators work. This allows us to create drastically different iteratees that work with the same enumerators:

    intsToStrings :: (Show a, Monad m) => Iteratee a m String
    intsToStrings = (unlines . map show) `fmap` consume

And then both of these lines work:

    run_ (getNumberEnum $$ sum6) >>= print
    run_ (getNumberEnum $$ intsToStrings) >>= print

## Exercises

* Write an enumerator that reads lines from stdin (as Strings). Make sure it works with this iteratee:

    <pre>printStrings :: Iteratee String IO ()
    printStrings = do
        mstring <- head
        case mstring of
            Nothing -> return ()
            Just string -> do
                liftIO $ putStrLn string
                printStrings</pre>

* Write an enumerator that does the same as above with words (ie, delimit on any whitespace). It should work with the same Iteratee as above.

* Do proper error handling in the getNumberEnum function above when the string is not a proper integer.

* Modify getNumberEnum to pull its input from a file instead of stdin.

* Use your modified getNumberEnum to sum up the values in two different files.

## Summary

* An enumerator is a **step transformer**: it feeds data into an iteratee to produce a new iteratee with an updated state. 

* Multiple enumerators can be fed into a single iteratee, and we finally use the run and run_ functions to extract results.

* We can use the $$, >>== and ==<< operators to apply an enumerator to an iteratee.

* When writing an enumerator, we only feed data to an iteratee in the Continue state; Yield and Error already represent final values.
