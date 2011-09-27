I would like to start a new campaign, trying to break Yesod. I want us to find every bug, every quirk, every bad API choice, every piece of badly performing code. At the end of the day, Yesod should be a bloody mess on the floor. (Either that, or there just aren't any bugs in it at all... yeah, right.) And not just Yesod: I want us to shake down hamlet, persistent, authenticate, clientsession, everything. Let's even see if we can find bugs in underlying libraries, and why the hell not, let's attack GHC as well.

I don't plan on adding any new features before a 1.0. That doesn't mean Yesod is feature-complete: I'd like better <abbr title="internationalization">i18n</abbr> support, more persistent backends, some more high-level Javascript integration. And I'm sure for every feature I can think of that could be added, the community could come up with 100. But that's not my focus right now. Rigth now, I want to fix things.

From my side, the first major step to take is to write a thorough test suite for Yesod. (Thanks for Greg Weber for reminding me that this is sorely lacking.) For example, the 0.6 release had a bug in getMessage. That should have been caught before I even hit <code>git commit</code>. Unfortunately, there don't seem to be many tools to help in this quest.

There is now a [web development Special Interest Group](http://www.haskellers.com/teams/4/) on Haskellers, and a [discussion on testing tools](http://www.haskellers.com/topics/4/). I strongly encourage everyone interested in this to join the group, follow [the news feed](http://www.haskellers.com/feed/team/4/) and participate in the discussion.

## Something More Concrete

For those of you looking for something concrete to get started on now, there's a code patch I'd like the community's help in reviewing. Matt Brown has made a [sorely needed commit](https://github.com/softmechanics/yesod/commit/f3afdfeea2b2cac40051ce1ad795e798faf96be6) which transforms the Route associated data type from a type family to a data family. If you don't know what that means, it basically gets rid of some dreaded "not injective" compiler errors and let's us drop some unneeded parameters in a few places.

I'd love to apply this patch immediately, but I have a small problem: I wrote a patch like this before that started causing a bug in GHC to pop up (only GHC 6.12.1 if I remember correctly). I have not been able to reproduce that bug with Matt's code, but if anyone in the community experiences a problem, please let me know sooner rather than later.

I'm going to give till about Monday night (UTC+2) to hear back on this. If no one complains, I'll assume the code works perfectly. I'd also appreciate hearing success stories, so that I know I'm not just talking to thin air on this. If it works, this patch will introduce a breaking change, requiring a major version bump. This is a major enough change that I wouldn't feel comfortable including it in the 1.0 release, so we'd likely be looking at a Yesod 0.7. However, the code change is minor enough that almost all 0.6 code will compile again without a hitch.

## Conclusion

I hope everyone will join me in this whack-a-bug frenzy on Yesod. Combined with some major documentation writing (and proofreading), a bit of performance tuning, and people complaining about bad API decisions, I think we can have a Yesod 1.0 release in the next few months that will meet the highest standards.
