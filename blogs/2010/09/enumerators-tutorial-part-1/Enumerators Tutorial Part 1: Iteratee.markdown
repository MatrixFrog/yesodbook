
This content is now part of the [Yesod book](http://www.yesodweb.com/book/enumerator). It is recommended to read there, since the content is more up-to-date.

## Introduction

One of the upcoming patterns in Haskell is the enumerators. Unfortunately, it's very difficult to get started with them since:

* There are multiple implementations, all with slightly different approaches.

* Some of the implementations (in my opinion) use incredibly confusing naming.

* The tutorials that get written usually don't directly target an existing implementation, and work more on building up intuition than giving instructions on how to use the library.

I'm hoping that this tutorial will fill the gap a bit. I'm going to be basing this on the [enumerator package](http://hackage.haskell.org/package/enumerator). I'm using version 0.4.0.2, but this should be applicable to older and hopefully newer versions as well. This package is newer and less used than the [iteratee](http://hackage.haskell.org/package/iteratee) package, but I've chosen it for three reasons:

* It has a much smaller dependency list.

* It's a smaller package, and therefore easier to wrap your mind around.

* I think the naming is better.

That said, both packages are built around the same basic principles, so learning one will definitely help you with the other.

### Three Parts

The title of this post says this is part 1. In theory, there will be three parts (though I may do more or less, I'm not certain yet). There are really three main concepts to learn to use the enumerator package: iteratees, enumerators and enumeratees. A basic definition would be:

* Iteratees are *consumers*: they are fed data and do something with it.

* Enumerators are *producers*: they feed data to an iteratee.

* Enumeratees are *pipes*: they are fed data from an enumerator and then feed it to an iteratee.

### What good are enumerators?

But before you really get into this library, let's give some motivation for *why* we would want to use it. Here's some real life examples I use the enumerator package for:

* When reading values from a database, I don't necessarily want to pull all records into memory at once. Instead, I would like to have them fed to a function which will consume them bit by bit.

* When processing a YAML file, instead of reading in the whole structure, maybe you only need to grab the value of one or two records.

* If you want to download a file via HTTP and save the results in a file, it would be a waste of RAM to store the whole file in memory and then write it out. Enumerators let you perform interleaved IO actions easily.

A lot of these problems can also be solved using lazy I/O. However, lazy I/O is not necessarily a panacea: you might want to read some of [Oleg's stuff](http://okmij.org/ftp/Streams.html#iteratee) on the pitfalls of lazy I/O.

## Intuition

*Note: you can see the code in this post as a [github gist](http://gist.github.com/605826).*

Let's say we want to write a function that sums the numbers in a list. Forgetting uninteresting details like space leaks, a perfectly good implementation could be:

    sum1 :: [Int] -> Int
    sum1 [] = 0
    sum1 (x:xs) = x + sum1 xs

But let's say that we don't have a list of numbers. Instead, the user is typing numbers on the command line, and hitting "q" when done. In other words, we have a function like:

    getNumber :: IO (Maybe Int)
    getNumber = do
        x <- readLine
        if x == "q"
            then return Nothing
            else return $ Just $ read x

We could write our new sum function as:

    sum2 :: IO Int
    sum2 = do
        maybeNum <- getNumber
        case maybeNum of
            Nothing -> return 0
            Just num -> do
                rest <- sum2
                return $ num + rest

It's fairly annoying to have to write two completely separate sum functions just because our data source changed. Ideally, we would like to generalize things a bit. Let's start by noticing a similarity between these two functions: they both only **yield** a value when they are informed that there are no more numbers. In the case of sum1, we check for an empty list; in sum2, we check for Nothing.

## The Stream datatype.

The first datatype defined in the enumerator package is:

    data Stream a = Chunks [a] | EOF

The EOF constructor indicates that no more data is available. The Chunks constructor simply allows us to put multiple pieces of data together for efficiency. We could now rewrite sum2 to use this Stream datatype:

    getNumber2 :: IO (Stream Int)
    getNumber2 = do
        maybeNum <- getNumber -- using the original getNumber function
        case maybeNum of
            Nothing -> return EOF
            Just num -> return $ Chunks [num]

    sum3 :: IO Int
    sum3 = do
        stream <- getNumber2
        case stream of
            EOF -> return 0
            Chunks nums -> do
                let nums' = sum nums
                rest <- sum3
                return $ nums' + rest

Not that it's much better than sum2, but at least it shows how to use the Stream datatype. The problem here is that we still refer explicitly to the getNumber2 function, hard-coding the data source.

One possible solution is to make the data source an argument to the sum function, ie:

    sum4 :: IO (Stream Int) -> IO Int
    sum4 getNum = do
        stream <- getNum
        case stream of
            EOF -> return 0
            Chunks nums -> do
                let nums' = sum nums
                rest <- sum4 getNum
                return $ nums' + rest

That's all well and good, but let's pretend we want to have *two* datasources to sum over: values the user enters on the command line, and some numbers we read over an HTTP connection, perhaps. The problem here is one of **control**: sum4 is running the show here by calling getNum. This is a **pull** data model. Enumerators have an **inversion of control/push** model, putting the enumerator in charge. This allows cool things like feeding in multiple data sources, and also makes it easier to write enumerators that properly deal with resource allocation.

## The Step datatype

So we need a new datatype that will represent the state of our summing operation. We're going to allow our operations to be in one of three states:

* Waiting for more data.

* Already calculated a result.

* For convenience, we also have an error state. This isn't strictly necessary (it could be modeled by choosing an EitherT kind of monad, for example), but it's simpler.

As you could guess, these states will correspond to three constructors for the Step datatype. The error state is modeled by <code>Error SomeException</code>, building on top of Haskell's extensible exception system. The already calculated constructor is:

    Yield b (Stream a)

Here, a is the *input* to our iteratee and b is the *output*. This constructor allows us to simultaneously produce a result and save any "leftover" input for another iteratee that may run after us. (This won't be the case with the sum function, which always consumes all its input, but we'll see some other examples that do no such thing.)

Now the question is how to represent the state of an iteratee that's waiting for more data. You might at first want to declare some datatype to represent the internal state and pass that around somehow. That's not how it works: instead, we simply use a function (very Haskell of us, right?):

    Continue (Stream a -> Iteratee a m b)

Euerka! We've finally seen the Iteratee datatype! Actually, Iteratee is a very boring datatype that is only present to allow us to declare cool instances (eg, Monad) for our functions. Iteratee is defined as:

    newtype Iteratee a m b = Iteratee (m (Step a m b))

This is important: **Iteratee is just a newtype wrapper around a Step inside a monad**. Just keep that in mind as you look at definitions in the enumerator package. So knowing this, we can think of the Continue constructor as:

    Continue (Stream a -> m (Step a m b))

That's much easier to approach: that function takes some input data and returns a new state of the iteratee. Let's see what our sum function would look like using this Step datatype:

    sum5 :: Monad m => Step Int m Int -- Int input, any monad, Int output
    sum5 =
        Continue $ go 0 -- a common pattern, you always start with a Continue
      where
        go :: Monad m => Int -> Stream Int -> Iteratee Int m Int
        -- Add the new input to the running sum and create a new Continue
        go runningSum (Chunks nums) = do
            let runningSum' = runningSum + sum nums
            -- This next line is *ugly*, good thing there are some helper
            -- functions to clean it up. More on that below.
            Iteratee $ return $ Continue $ go runningSum'
        -- Produce the final result
        go runningSum EOF = Iteratee $ return $ Yield runningSum EOF

The first real line (<code>Continue $ go 0</code>) initializes our iteratee to its starting state. Just like every other sum function, we need to explicitly state that we are starting from 0 somewhere. The real workhorse is the go function. Notice how we are really passing the state of the iteratee around as the first argument to go: this is also a very common pattern in iteratees.

We need to handle two different cases: when handed an EOF, the go function **must** Yield a value. (Well, it could also produce an Error value, but it definitely **cannot** Continue.) In that case, we simply yield the running sum and say there was no data left over. When we receive some input data via Chunks, we simply add it to the running sum and create a new Continue based on the same go function.

Now let's work on making that function a little bit prettier by using some built-in helper functions. The pattern <code>Iteratee . return</code> is common enough to warrant a helper function, namely:

    returnI :: Monad m => Step a m b -> Iteratee a m b
    returnI = Iteratee . return

So for example,

    go runningSum EOF = Iteratee $ return $ Yield runningSum EOF

becomes

    go runningSum EOF = returnI $ Yield runningSum EOF

But even *that* is common enough to warrant a helper function:

    yield :: Monad m => b -> Stream a -> Iteratee a m b
    yield x chunk = returnI $ Yield x chunk

so our line becomes

    go runningSum EOF = yield runningSum EOF

Similarly,

    Iteratee $ return $ Continue $ go runningSum'

becomes

    continue $ go runningSum'

## Monad instance for Iteratee

This is all very nice: we now have an iteratee that can be fed numbers from any monad and sum them. It can even take input from different sources and sum them together. (By the way, I haven't actually shown you how to feed those numbers in: that is in part 2 about enumerators.) But let's be honest: sum5 is an ugly function. Isn't there something easier?

In fact, there is. Remember how I said Iteratee really just existed to facilitate typeclass instances? This includes a monad instance. Feel free to look at the code to see how that instance is defined, but here we'll just look at how to use it:

    sum6 :: Monad m => Iteratee Int m Int
    sum6 = do
        maybeNum <- head -- not head from Prelude!
        case maybeNum of
            Nothing -> return 0
            Just i -> do
                rest <- sum6
                return $ i + rest

That head function is not from Prelude, it's from the Data.Enumerator module. Its type signature is:

    head :: Monad m => Iteratee a m (Maybe a)

which basically means give me the next piece of input if it's there. We'll look at this function in more depth in a bit.

Go compare the code for sum6 with sum2: they are amazingly similar. You can often build up more complicated iteratees by using some simple iteratees and the Monad instance of Iteratee.

## Interleaved I/O

Alright, let's look at a totally different problem. We want to be fed some strings and print them to the screen one line at a time. One approach would be to use lazy I/O:

    lazyIO :: IO ()
    lazyIO = do
        s <- lines `fmap` getContents
        mapM_ putStrLn s

But this has two drawbacks: 

* It's tied down to a single input source, stdin. This could be worked around with an argument giving a datasource.

* But let's say the data source is some scarce resource (think: file handles on a very busy web server). We have no guarantees with lazy I/O of when those file handles will be released.

Let's look at how to write this in our new high-level monadic iteratee approach:

    interleaved :: MonadIO m => Iteratee String m ()
    interleaved = do
        maybeLine <- head
        case maybeLine of
            Nothing -> return ()
            Just line -> do
                liftIO $ putStrLn line
                interleaved

The liftIO function comes from the transformers package, and simply promotes an action in the IO monad to any arbitrary MonadIO action. Notice how we don't really track any state with this iteratee: we don't care about its result, only its side effects.

## Implementing head

As a last example, let's actually implement the head function.

    head' :: Monad m => Iteratee a m (Maybe a)
    head' =
        continue go
      where
        go (Chunks []) = continue go
        go (Chunks (x:xs)) = yield (Just x) (Chunks xs)
        go EOF = yield Nothing EOF

Like our sum6 function, this also wraps an inner "go" function with a continue. However, we now have *three* clauses for our go function. The first handles the case of <code>Chunks []</code>. To quote the enumerator docs:

> (Chunks []) is used to indicate that a stream is still active, but currently has no available data. Iteratees should ignore empty chunks.

The second clause handles the case where we are given some data. In this case, we yield the first element in the list, and return the rest as leftover data. The third clause handles the end of input by returning Nothing.

## Exercises

* Rewrite sum6 using [liftFoldL'](http://hackage.haskell.org/packages/archive/enumerator/0.4/doc/html/Data-Enumerator.html#v:liftFoldL-39-).

* Implement the [consume](http://hackage.haskell.org/packages/archive/enumerator/0.4/doc/html/Data-Enumerator.html#v:consume) function using first the high-level functions like head, and then using only low-level stuff.

* Write a modified version of consume that only keeps every other value, once again using high-level functions and then low-level constructors.

## Next time

Well, now you can write iteratees, but they're not very useful if you can't actually use them. Next time we'll cover what an enumerator is, some basic enumerators included with the package, how to run these things and how to write your own enumerator.

## Summary

Here's what I consider the most important things to glean from this tutorial:

* Iteratee is a simple wrapper around the Step datatype to allow for cool typeclass instances.

* Using the Monad instance of Iteratee can allow you to build up complicated iteratees from simpler ones.

* The three states an enumerator can be in are Continue (still processing data), Yield (a result is ready) and Error (duh).

* Well behaved iteratees will never return a Continue after receiving an EOF.
