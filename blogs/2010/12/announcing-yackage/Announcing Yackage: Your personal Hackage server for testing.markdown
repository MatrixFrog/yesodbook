I was speaking with [my coworker Yitz](http://www.haskellers.com/user/Yitz_Gale/) about a project he's working on. Basically, he's going to end up with about 16 cabal packages that are not going to be deployed to Hackage, and wanted us to set up a Hackage server for our company to deploy these kinds of things. However, getting all the pieces of Hackage aligned properly for such a simple use case seemed a bit overkill.

I then realized that I had the exact same problem during Yesod development: before I make a major release, I usually end up with about 10-15 packages that are not yet live on Hackage. It gets to be a real pain when suddenly wai-extra is depending on network 2.3 and authenticate requires network 2.2, and suddenly I need to manually recompile 10 packages.

So I decided to write up a simple web service to act as a local Hackage server. It has no security (anyone with access can upload a package), doesn't build haddocks, doesn't show package descriptions, etc. All it does is:

* Show a list of uploaded packages/versions
* Links to the tarballs
* Allows you to upload new versions, which will automatically overwrite existing packages
* Provides the 00-index.tar.gz file needed by cabal-install, as well as the tarballs for all the packages

In order to use this, just do the following:

* cabal install yackage
* run "yackage"
* Upload your packages
* Add <code>remote-repo: yackage:http://localhost:3500/</code> to your ~/.cabal/config file
* cabal update
* Install your packages are usual

You'll need to leave yackage running whenever you want to run an update or download new packages. A few other usage notes:

* If you overwrite a package, your cache folder will still have the old version. You might want to just wipe our your cache folder on each usage.
* Running cabal update will download the update for both yackage *and* the main hackage server; the latter can be a long process depending on your internet connection.

Here's a little shell script that will disable the Hackage repo, wipe our the Yackage cache, update and re-enable the Hackage repo:

    #!/bin/sh

    CABAL_DIR=~/.cabal

    cp $CABAL_DIR/config $CABAL_DIR/config.sav
    sed 's/^remote-repo: hackage/--remote-repo: hackage/' < $CABAL_DIR/config.sav > $CABAL_DIR/config
    rm -rf $CABAL_DIR/packages/yackage
    cabal update
    cp $CABAL_DIR/config.sav $CABAL_DIR/config

I hope others find this tool useful.
