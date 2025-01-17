I got an email recently asking for some documentation on the [data-object-yaml](http://hackage.haskell.org/package/data-object-yaml). It's a fairly straight-forward package, with one or two surprises lurking inside, so I think a blog post ought to do it justice. If anyone has questions, just ask it in a comment and I'll try to address it.

As a bit of background, this package is built on a few other packages I wrote. yaml is a low-level wrapper around the C libyaml library, with an enumerator interface. data-object is a package defining a data type:

    data Object k v = Scalar v
                    | Sequence [Object k v]
                    | Mapping [(k, Object k v)]

In other words, it can represent JSON data fully, and YAML data almost fully. In particular, it doesn't handle cyclical aliases, which I hope doesn't really occur too much in real life.

Another package to deal with is failure: it basically replaces using an Either for error-handling into a typeclass. It has instances for Maybe, IO and lists by default.

The last package is convertible-text, which is a fork of John Goerzen's convertible package. The difference is it supports both conversions that are guaranteed to succeed (Int -> String) and ones which may fail (String -> Int), and also supports various textual datatypes (String, lazy/strict ByteString, lazy/string Text).

## YamlScalar and YamlObject

We have a <code>type YamlObject = Object YamlScalar YamlScalar</code>, where a YamlScalar is just a ByteString value with a tag and a style. A "style" is how the data was represented in the underlying YAML file: single quoted, double quoted, etc.

Then there is an IsYamlScalar typeclass, which provides fromYamlScalar and toYamlScalar conversion functions. There are instances for all the "text-like" datatypes: String, ByteString and Text. The built-in instances all assume a UTF-8 data encoding. And around this we have toYamlObject and fromYamlObject functions, which do exactly what they sound like.

## Encoding and decoding

There are two encoding files: encode and encodeFile. You can guess the different: the former produces a ByteString (strict) and the latter writes to a file. They both take an Object, whose keys and values must be an instance of IsYamlScalar. So, for example:

    encodeFile "myfile.yaml" $ Mapping
        [ ("Michael", Mapping
            [ ("age", Scalar "26")
            , ("color", Scalar "blue")
            ])
        , ("Eliezer", Mapping
            [ ("age", Scalar "2")
            , ("color", Scalar "green")
            ])
        ]

decoding is only slightly more complicated, since the decoding can fail. In particular, the return type is an IO wrapped around a Failure. For example, you could use:

    maybeObject <- decodeFile "myfile.yaml"
    case maybeObject of
        Nothing -> putStrLn "Error parsing YAML file."
        Just object -> putStrLn "Successfully parsed."

If you just want to throw any parse errors as IO exception, you can use join:

    import Control.Monad (join)
    object <- join $ decodeFile "myfile.yaml"

This takes advantage of the IO instance of Failure.

## Parsing an Object

In order to pull the data out of an Object, you can use the helper functions from Data.Object. For example:

    import Data.Object
    import Data.Object.Yaml
    import Control.Monad

    main = do
        object <- join $ decodeFile "myfile.yaml"
        people <- fromMapping object
        michael <- lookupMapping "Michael" people
        age <- lookupScalar "age" michael
        putStrLn $ "Michael is " ++ age ++ " years old."

## And that's it

There's really not more to know about this library. Enjoy!
