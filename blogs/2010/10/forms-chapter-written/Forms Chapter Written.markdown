Special service announcement: please fill out the [Haskellers.com survey](https://spreadsheets.google.com/viewform?formkey=dG8tcVhBdzRSS1lkdVV6U281emtRMWc6MQ). That is all ;).

I just wanted to give a heads-up that the [forms chapter of the Yesod book](http://docs.yesodweb.com/book/forms/) is up. This is still mostly un-proofread, but contains a good amount of information and should be enough to get you started on forms. I'd appreciate feedback on where it could be further clarified. (By the way, that goes for the other chapters as well.)

## Schrödinger's applicative cat

While going over this chapter with my wife, I was trying to figure out a good analogy for applicatives. I'm not sure if I just explained why [monads are like burritos](http://byorgey.wordpress.com/2009/01/12/abstraction-intuition-and-the-monad-tutorial-fallacy/), but it's cute enough to put up anyway.

We all know about evil Mr. Schrödinger (is it Dr. Schrödinger?) who locked his cat in a bag with some poison. We're not sure if at any point in time the cat is alive or dead. (Hint: this is like a Maybe.) So in Haskell terms, we might have:

    data Cat = Cat { pounds :: Int }
    data Bag a = Dead | Alive a

Now, we're not sadists here (besides locking cats up in their bags with poison), so we decide we would like to feed this perhaps-living cat. So we write up our little cat-feeding function:

    data Mouse = Mouse
    feedCat :: Mouse -> Cat -> Cat
    feedCat Mouse (Cat oldWeight) = Cat $ oldWeight + 1

But time comes to feed the cat, and we realize that our cat lives in a bag! In other words, our cat (call him whiskers) is:

    whiskers :: Bag Cat

We need some way to send the food into the bag. We'll call that intoBag:

    intoBag :: (a -> b) -> Bag a -> Bag b
    intoBag f Dead = Dead
    intoBag f (Alive a) = Alive $ f a

And now we can feed whiskers with:

    intoBag (feedCat Mouse) whiskers

All is well and good, until we realize Mr. Schrödinger decided he wanted to put the mouse in a bag as well:

    squeaker :: Bag Mouse

We try out intoBag function, and realize the only thing we can do is this:

    intoBag feedCat squeaker :: Bag (Cat -> Cat)

We need some way of feeding a mouse that may be alive to a cat that may be alive. Firstly, let's think about the outcome of these things:

* If both the mouse and the cat are alive, then the cat can eat the mouse. The cat will then still be alive. (You can guess what happens to the mouse.)

* If the cat is dead, well, the cat's dead.

* If the mouse is dead, the cat doesn't have any food, so he dies too.

Fairly gruesome. But we still haven't figured out a way to do this! Turns out we need a combineBags function:

    combineBags :: Bag (a -> b) -> Bag a -> Bag b
    combineBags (Alive f) (Alive a) = Alive $ f a
    combineBags _ _ = Dead

And then we can feed whiskers:

    intoBag feedCat squeaker `combineBags` whiskers

Assuming, of course, he's still alive.
