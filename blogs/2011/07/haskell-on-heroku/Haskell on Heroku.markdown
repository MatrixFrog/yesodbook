Deploying Yesod keeps getting better. I just deployed an application to Heroku. Heroku isn't right for every usage case, but it is an easy and free (for one small app) deployment option that we should take advantage of.

Heroku recently created a new Cedar application stack that is very flexible- it was just a matter of time before Haskell was deployed there. Big thanks goes to Mark Wotton and everyone else at the recent Australian Haskell hackathon for [demonstrating](https://github.com/mwotton/heroku_haskell_demo) you can deploy Haskell to Heroku- they got an app running, although they didn't have a real database working.

I deployed a simple landing page Yesod application to Heroku. It saved e-mails to a database- demonstrating a web-app using the Heroku shared Postgresql database. And I know it works because I did a `heroku db:pull` to copy the database to my local machine. This is why people use Heroku- they provide a lot of nice built in features and take the hassle out of deploying. The other nice thing they do is aggregate logs. `heroku logs` will show your recent log output, and log aggregation and rotation is already setup for you.

As part of this effort I started a [heroku package](http://hackage.haskell.org/package/heroku-0.1) containing code for deploying to Heroku. All the package does now is parse the DATABASE_URL environment variable to help with connecting to the database.

# Configuring Yesod

I also worked on Yesod to make it much more configurable. Previously we just had a production build flag, and all application settings were determined at compile time. Static guarantees are nice, but this was a very inflexible setup that isn't usable for Heroku. You now have the following tools at your disposal for configuring your application:

* YAML configuration files
* Command line arguments
* Check what environment the application is in at runtime
* Continue to check what environment the application is at build time with CPP flags.

These aren't things we are comfortable direcly coupling to Yesod, so this is all generated in the new scaffolding that will be in the 0.9 release. Or you can grab the latest Yesod [from github](https://github.com/yesodweb/yesod). The downside is that the upgrade path is unclear. You might have different ideas as to what you want- so for now you have to generate a new scaffold site and look at the config/Settings.hs and the main file and decide what you want. We plan on provide a migration guide for the latest version of Yesod that will provide some more specific instructions.

In the new scaffold, command line arguments override existing settings. There is still a config/Settings.hs but also a settings.yaml file with a few site settings. Database connectiion parameters are in YAML files named after their database type (config/postgres.yml).

The scaffolder also generates a [Heroku Procfile](https://github.com/yesodweb/yesod/blob/master/yesod/scaffold/deploy/Procfile.cg), which documents what you need to do to deploy in its comments. It actually only contains one line of Heroku configuration: 

    web: ./dist/build/~project~/~project~ -p $PORT

Yesod is now setup by default to accept the port command line argument. There are a few other simple steps to go through for your application. There is one big issue that might prevent you from deploying: you must compile your app on 64bit and statically link it. I did this on my Ubuntu computer without issue for my application by just adding the `-static` ghc option. However, I am told that static linking is not always this simple and fool proof.

What we have now is a pure Warp deployment. This means no Apache/Nginx for static file serving. But this doesn't concern us- Warp can serve static assets quickly, and in Yesod 0.9 static files will automatically be given proper caching headers.

If you already work on 64 bit Linux and are using postgresql, Heroku is likely the easiest way to deploy your next application. 

The next Yesod release (0.9) will be the easiest web application to deploy available anywhere- a single binary will correclty serve your dynamic and static requests, use every core, and handle IO asynchronously. We just need a few recipes for deploying to different environments. Let us know what your deployment techniques are.


# Update: Heroku is not just for Ruby

Someone on Reddit asked for more particulars. Heroku's new cedar stack is designed to be flexible enough to run Ruby, Node.js, and Clojure. This new flexibility means more supported languages. However, it also means you can run any executable on Heroku. The issue is that you can't *install* to Heroku. In interpreted languages you don't have to do anything special to install packages- you just need the interpreter to be installed. In a compiled language you must staticly link your binary.

Perhaps the biggest downside of static linking is the long time it takes to transfer a new binary. Anyone have solutions for quicker binary updates?