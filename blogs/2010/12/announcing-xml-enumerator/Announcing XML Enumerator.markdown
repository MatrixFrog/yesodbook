I'm very happy to announce the first release of the [xml-enumerator](http://hackage.haskell.org/package/xml-enumerator) package. But first, a word from our sponsors: a large portion of the code in this package was written for work at my day job with [Suite Solutions](http://www.suite-sol.com/). We are a company that focuses on documentation services, especially using XML in general and DITA in particular. The company has graciously agreed to allow me to release this code as open source.

For those who are not yet aware, I'm a very big fan of John Millikan's [enumerator](http://hackage.haskell.org/package/enumerator) package, and so it should come as no surprise that when I needed to pick an XML parsing library I turned to his work. I was surprised to find not one, but *two* librarys: libxml-enumerator and expat-enumerator, which each wrapped around the respective C libraries. libxml-enumerator is very difficult to get working on Windows systems, and expat-enumerator does not have support for XML namespaces or doctypes. (That may be fixed in the future.)

What's nice is that both of these packages rely upon the same datatypes, provided by the [xml-types](http://hackage.haskell.org/package/xml-types) package (also written by John), and therefore can be mostly swapped out transparently. Unfortunately, there was no support for rendering XML using these datatypes, which was something I needed for the tool I was writing. As such, xml-enumerator is composed of three pieces:

* A native XML parser based on attoparsec.
* An XML renderer based on blaze-builder.
* A set of parser combinators I developed for the Yesod book.

Let me give a brief introduction to each of these:

## The parser

The parseBytes function provides an enumeratee to convert a stream of ByteStrings into a stream of Events. It follows mostly the same type signature as the parseBytesIO functions provided by libxml-enumerator and expat-enumerator, but since it does not call foreign code can live in any monad. There is one limitation: it requires the input to be in UTF8 encoding.

To overcome this, the Text.XML.Enumerator.Parse module also provides a detectUtf enumeratee, which will automatically convert UTF-16/32 (big and little endian) into UTF8, leaving UTF8 data unchanged. Therefore, to get a list of Events from a file, you could use:

    run_ $ enumFile "myfile.xml" $$ joinI $ detectUtf $$ joinI $ parseBytes $$ consume

Ideally, I would have liked to make this parser work on a stream of Texts instead of ByteStrings, but unfortunately there is not yet an equivalent of attoparsec for Texts.

## The renderer

The renderer is simply two enumeratees: renderBuilder converts a stream of Events into a stream of Builders (from the blaze-builder package). renderBytes uses renderBuilder and the blaze-builder-enumerator package to convert a stream of Events into a stream of optimally-sized ByteStrings, involving minimal buffer copying. In order to re-encode an XML file, for example, we could run:

    withBinaryFile "output.xml" WriteMode $ \h ->
        run_ $ enumFile "input.xml" $$ joinI $ detectUtf $$ joinI $ parseBytes
            $$ joinI $ renderBytes $$ iterHandle h

Note that you could equally well use libxml-enumerator or expat-enumerator for the parsing, since they are all using the same Event type.

## Parser combinators

The combinator library is meant for parsing simplified XML documents. It uses the simplify enumeratee to convert a stream of Events into SEvents. This essentially throws out all events besides element begin and end events, converts entities into text based on a function you provide it, and concatenates together adjacent contents.

Attribute parsing is handled via a AttrParser monad, which lets you deal with both required and optional attributes and whether parsing should fail when unexpected attributes are present. Tag parsing has three functions (tag, tag' and tag''... I'm horrible at naming things) that go from most-general to easiest-to-use, and there are two content parsing functions.

There are three combinators: choose selects the first successful parser, many runs a parser until it fails, and force requires a parser to run successfully. And finally, there are two convenience functions, parseFile and parseFile_, that automatically uses detectUtf, parseBytes and simplify for you. The Text.XML.Enumerator.Parse module begins with this sample program:

     {-# LANGUAGE OverloadedStrings #-}
     import Text.XML.Enumerator.Parse
     import Data.Text.Lazy (Text, unpack)
     
     data Person = Person { age :: Int, name :: Text }
         deriving Show
     
     parsePerson = tag' "person" (requireAttr "age") $ \age -> do
         name <- content'
         return $ Person (read $ unpack age) name
     
     parsePeople = tag'' "people" $ many parsePerson
     
     main = parseFile_ "people.xml" (const Nothing) $ force "people required" parsePeople

Given a people.xml file containing:

     <?xml version="1.0" encoding="utf-8"?>
     <people>
         <person age="25">Michael</person>
         <person age="2">Eliezer</person>
     </people>

produces:

    [Person {age = 25, name = "Michael"},Person {age = 2, name = "Eliezer"}]

## Relation to some recent work

If you are wondering, the release of this package is very much related to some stuff I mentioned in [my last blog post](http://docs.yesodweb.com/blog/serendipity-wai-http-enumerator/). In particular, I wrote a first version of blaze-builder-enumerator while working on the renderer for this package, and was working on demonstrating this package as a web service when I decided to stitch together WAI and xml-enumerator.

## Conclusion

Since this is an initial release, there are very possibly some bugs in it, so please let me know if you find anything wrong. And I'd just like to give another thank you to my employers for contributing this to the community.
