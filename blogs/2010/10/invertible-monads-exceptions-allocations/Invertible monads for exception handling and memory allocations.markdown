
## tldr

I've written a MonadInvertIO typeclass, instances for a bunch of the standard monad transformers, and a full test suite. It's currently living in my neither github repo, but after some more testing I'll probably release it as its own package. You can see the [code](http://github.com/snoyberg/neither/blob/master/Control/Monad/Invert.hs) and [tests](http://github.com/snoyberg/neither/blob/master/runtests.hs).

The rest of this post describes the problems with MonadCatchIO, builds motivation for a new solution through the identity, writer, error and reader monads, and finally [presents a new approach](#monadinvertio). You should feel free to skip around however much you want.

## The problem

I was rather dismayed to see Haskellers spitting out PoolExhaustedExceptions a few days after launch. That exception gets thrown when trying to allocate a database connection from the connection pool. I knew it wasn't simply setting the connection pool size too small: once the errors started, they continued until I did a hard process restart. Rather, connections were leaking.

I wrote a bunch of debug code to isolate the line where the connections were being allocated and not returned (the code is still live on Haskellers, just grep the codebase for debugRunDB), and eventually I traced it to a line looking like this:

    runDB $ do
        maybeUserName <- isThereAUserName
        case maybeUserName of
            Just username -> lift $ redirect RedirectTemporary $ UserR username
            Nothing -> return ()

To the uninitiated: run a database action to see if the user has a username, and if so redirect to his/her canonical URL. After a little tracing, I realized the issue was this:

* Yesod uses a specialized Error monad to allow short-circuiting, for example in cases of redirecting.

* The Persistent package uses the MonadCatchIO-transformer's finally function to ensure database connections are returned to the pool, even in the presence of exceptions.

* The MonadCatchIO family of packages are completely broken :(. For example, finally correctly handles exceptions and normal return, but never calls cleanup code on short-circuiting (ie, calling throwError).

I wrote an [email to the cafe](http://www.mail-archive.com/haskell-cafe@haskell.org/msg82859.html) explaining the flaws in MonadCatchIO in detail. Essentially, one was to fix this is to create a new typeclass for the bracket function. However, it's always bothered me that we need to write all of these special functions to handle MonadIOs: it seems that it should be possible to simply "invert" the monads. The rest of this post discusses that inversion.

## Inverting IdentityT

Whenever trying to write something generalized for a whole bunch of monads, it's great to start with IdentityT. Let's say we have an action and some cleanup code:

    action :: IdentityT IO ()
    action = liftIO (putStrLn "action") >> error "some error occured"
    
    cleanup :: IdentityT IO ()
    cleanup = liftIO $ putStrLn "cleanup"

We want to run action, and then regardless of the presence of exceptions, call cleanup. This is a perfect use case for finally, but in Control.Exception it's defined as:

    finally :: IO a -> IO b -> IO a

In order to use this, we need to force IdentityT IO to look like a IO. Well, in this case, it's rather straight-forward.

    runSafely = IdentityT $
        runIdentityT action `finally` runIdentityT cleanup

Fairly simple: unwrap the IdentityT constructor, call finally, and then wrap it up again.

## What about WriterT?

Well, inverting a monad that does nothing is not very impressive. Let's try it out on WriterT:

    actionW, cleanupW :: WriterT [Int] IO ()
    actionW = liftIO (putStrLn "action") >> tell [1]
    cleanupW = liftIO (putStrLn "cleanup") >> tell [2]
    runSafelyW = WriterT $ runWriterT actionW `finally` runWriterT cleanupW

That turned out to be incredibly easy. Let's see what happens when we run this:

    > runWriterT runSafelyW >>= print
    action
    cleanup
    ((),[1])

Wait a second, what about <code>tell [2]</code>? Well, the return value from the cleanup argument to finally gets ignored, and therefore so does anything we <code>tell</code> in there. This becomes more obvious if we expand the calls to tell:

    actionW' = WriterT (putStrLn "action" >> return ((), [1]))
    cleanupW' = WriterT (putStrLn "cleanup" >> return ((), [2]))

    runSafelyW' = WriterT $ runWriterT actionW' `finally` runWriterT cleanupW'
    -- the same as
    runSafelyW'' = WriterT $ finally
        (runWriterT $ WriterT (putStrLn "action" >> return ((), [1])))
        (runWriterT $ WriterT (putStrLn "cleanup" >> return ((), [2])))
    -- runWriterT . WriterT == id, so this reduces to
    runSafelyW''' = WriterT $ finally
        (putStrLn "action" >> return ((), [1]))
        (putStrLn "cleanup" >> return ((), [2]))

This may or may not be what you want (an argument could be made either way), but let's see the next example before you decide that this is the wrong approach.

## ErrorT

It turns out the code here is nearly identical again:

    actionE, cleanupE :: ErrorT String IO ()
    actionE = do
        liftIO $ putStrLn "action1"
        throwError "throwError1"
        liftIO $ putStrLn "action2"
    cleanupE = liftIO (putStrLn "cleanup") >> throwError "throwError2"
    runSafelyE = ErrorT $ runErrorT actionE `finally` runErrorT cleanupE

As a quick refresher: throwError "short-circuits" the remainder of the computation. For example, "action2" will never be printed. So what's the output of this thing

    > runErrorT runSafelyE >>= print
    action1
    cleanup
    Left "throwError1"

The "throwError2" never shows up, but the cleanup does. Just to stress the point, let's modify this ever so slightly and remove the first throwError:

    actionE2 :: ErrorT String IO ()
    actionE2 = do
        liftIO $ putStrLn "action1"
        liftIO $ putStrLn "action2"
    runSafelyE2 = ErrorT $ runErrorT actionE2 `finally` runErrorT cleanupE

This time, the output is:

    > runErrorT runSafelyE2 >>= print
    action1
    action2
    cleanup
    Right ()

Wait a second: why didn't I get a throwError2 *this* time?  Once again, this has to do with ignoring return values from a cleanup function. Let's desugar again and look at what's happening under the surface:

    runSafelyE' = ErrorT $ finally
        (do
            a <- fmap Right $ putStrLn "action1"
            case a of
                Left e -> return $ Left e
                Right a' -> fmap Right $ putStrLn "action2"
        )
        (do
            a <- fmap Right $ putStrLn "cleanup"
            case a of
                Left e -> return $ Left e
                Right a' -> return $ Left "throwError2"
        )

This is just a straight mechanical translation of runSafelyE2 using the definition of ErrorT. We can now remove some of the noise, since we know at compile time whether we are returning Right or Left:

    runSafelyE'' = ErrorT $ finally
        (do
            putStrLn "action1"
            putStrLn "action2"
            return $ Right ()
        )
        (do
            putStrLn "cleanup"
            return $ Left "throwError2"
        )

We can see that the throwError2 is simply a result of the return value of the cleanup function, which gets ignored by finally. Once again, like in WriterT, it seems that our abstraction is destroying the power of our monads. This is arguably true, but it's also the only correct way to deal with the situation. Let's say that we changed our action function to now be:

    (do
        putStrLn "action1"
        putStrLn "action2"
        returnsLeftOrRight :: IO (Either String ())
    )

If returnsLeftOrRight results in <code>Left "throwError3"</code>, then we would want our function to print action1, action2, cleanup and result in Left "throwError3". In order to call the cleanup function at all, however, it can't be dependent on the return value of action. This is exactly where MonadCatchIO failed: over there, if the action returns a Left, it short-circuits the cleanup function from running.

In theory, you could make the argument that we should follow this logic train:

* If the action returns Left, run through the cleanup code and return the action's return value, ignoring the cleanup return value.

* Otherwise, if the cleanup code returns Left, return the cleanup code's Left.

* Otherwise, return the action's Right value.

But this is adding a lot of complexity, and it seems to me to work against us. We've seen now with both WriterT and ErrorT that the obvious definitions simply ignore the result of the cleanup function, just like when you use finally in the IO monad. This is the way I've implemented my code, and leads to one important caveat:

__Excluding IO effects themselves, never rely on monadic side effects from cleanup code.__

This may seem to make the whole exercise futile, but I think the majority of the time when we would want to use this approach, it is simply to perform cleanup that must occur in the IO monad.

## ReaderT

I've always considered the reader monad to be the simplest of the monads. I find it ironic that reader (and state) introduce a major complication in our approach. We'll define out action and cleanup pretty much as before:

    actionR, cleanupR :: ReaderT String IO ()
    actionR = ask >>= liftIO . putStrLn
    cleanupR = liftIO $ putStrLn "cleanup"

Previously, we had <code>runWriterT actionW :: IO (...)</code> and <code>runErrorT actionE :: IO (...)</code>. However, reader doesn't work that way; instead: <code>runReaderT actionR :: r -> IO (...)</code>. So we have to slightly modify our runSafelyR function to deal with the parameter:

    runSafelyR = ReaderT $ \r ->
        runReaderT actionR r
        `finally`
        runReaderT cleanupR r

For our little case here, it's pretty simple to account for this extra parameter. It's a little bit more complicated in the state monad (which I won't cover here, I've bored you enough already), but the point where it really bites is in MonadInvertIO itself.

<h2 id="monadinvertio">MonadInvertIO</h2>

Hopefully the above four monad examples showed that it's entirely possible to turn a transformer inside out. If this makes sense for a single layer (a wraps b becomes b around a), it should makes sense for any number of layers (abc becomes cba). This is similar to the concept of the MonadTrans and MonadIO typeclasses: the former allows us to lift one level, while the latter lifts all the way to IO.

I originally started with the same premise of having a MonadInvert typeclass to invert one monadic layer and MonadInvertIO to invert to the IO layer, but due to technicalities it would be tedious to work this way. Plus, I'm not sure if there is much use for this functionality outside the realm of MonadInvertIO. In any event, the typeclass is:

    class Monad m => MonadInvertIO m where
        data InvertedIO m :: * -> *
        type InvertedArg m
        invertIO :: m a -> InvertedArg m -> IO (InvertedIO m a)
        revertIO :: (InvertedArg m -> IO (InvertedIO m a)) -> m a

This is using type families. InvertedIO gives the "inverted" representation of our monad, and InvertedArg gives the argument to our function. This argument is the complication I alluded to in the reader section above. For monads like error and writer, InvertedArg is not necessary.

invertIO takes our monadic value and returns a function that takes our InvertedArg and returns something in the IO monad. This should look familiar from the reader section above. revertIO simply undoes that action.

As an easy example, let's see the IO instance:

    instance MonadInvertIO IO where
        newtype InvertedIO IO a = InvIO { runInvIO :: a }
        type InvertedArg IO = ()
        invertIO = const . liftM InvIO
        revertIO = liftM runInvIO . ($ ())

In this case, InvertedIO doesn't do anything, which isn't really surprising (this instance is, after all, just a wrapper). We don't need any arguments, so InvertedArg is (). invertIO and revertIO are also just dealing with the typing requirements.

To get a better idea of how these things work, let's look at IdentityT:

    instance MonadInvertIO m => MonadInvertIO (IdentityT m) where
        newtype InvertedIO (IdentityT m) a =
            InvIdentIO { runInvIdentIO :: InvertedIO m a }
        type InvertedArg (IdentityT m) = InvertedArg m
        invertIO = liftM (fmap InvIdentIO) . invertIO . runIdentityT
        revertIO f = IdentityT $ revertIO $ liftM runInvIdentIO . f

IdentityT itself does not add anything to the representation of the data, but the monads underneath it might. Therefore, its InvertedIO associated type references the InvertedIO of the underlying monad. We do the same with InvertedArg. invertIO and revertIO simply do some wrapping and unwrapping.

Some of the instances can get a bit tricky, but it's all built on these principles. Feel free to [explore the code](http://github.com/snoyberg/neither/blob/master/Control/Monad/Invert.hs), I don't want anyone reading this post to die of boredom.

## Using the typeclass

The final piece in the puzzle is how to actually use this typeclass in real life. A great, straightforward example is a new definition of finally:

    import qualified Control.Exception as E
    finally :: MonadInvertIO m => m a -> m b -> m a
    finally action after = revertIO $ \a ->
        invertIO action a
        `E.finally`
        invertIO after a

This is the general model for all uses of the library. We need to pair up revertIO and invertIO. Like in the reader example, we end up with an argument (called a here), which we need to pass around every time we call invertIO. Once we've done that, we now have two actions living purely in the IO monad, so we can use E.finally on them.

In addition to exception handling, we can use this approach for memory allocation:

    alloca :: (Storable a, MonadInvertIO m) => (Ptr a -> m b) -> m b
    alloca f = revertIO $ \x -> A.alloca $ flip invertIO x . f

As I said at the beginning, I have a test suite running to ensure the inversion code is working correctly. I'm going to test it out in more complicated situations, but I believe this could be used as a general solution to the recurring problem of functions requiring IO-specific arguments.

I'd appreciate feedback on this idea, *especially* if you find anything which seems to be flawed. I don't want to produce another broken finally function.
