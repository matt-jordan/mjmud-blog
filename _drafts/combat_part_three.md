---
layout: post
title: "Combat Part 3: A bag of dice"
---

Many years ago, I read a great post on how random numbers are bad for games, particularly role-playing games. I can't find it, but if I ever do, I'll come back and link it here because it did a great job of explaining the concept.

I'll try to summarize.

There's at least some [statements floating around](https://github.com/ckknight/random-js) that the implementation we have for picking between two integer values in [randomInteger]() will not create a uniform distribution. For where I'm using that right now - picking an exit in a room, deciding if a monster will wander - that's probably not that big of a deal. When determining combat results however, you really don't want to bias towards hitting too often or missing too often. So that's one consideration.

Another consideration is simply that truly random is *bad* for games. No one wants to roll a '1' 20 times in a row, but that is perfectly possible with a large enough sample size. We want to limit the number of potential bad and good rolls that can happen, while still keeping things 1/ uniformly distributed and 2/ semi-random.

Consider, instead, a dice bag. For our purposes, we'll say we're rolling six sided dice, and we have 36 dice. With a uniform distribution, we'd expect six instances of each value: 6 1's, 6 2's, etc. We then put all the dice in a bag and shake it up.

While it's still possible to pull 6 1's in a row, once we've exhausted them out of the bag we're done with the 1's. We can't roll a critical failure again until we've exhausted the other 30 dice. That gives the character some reprieve from a non-infinite-but-very-long string of bad luck.

So let's make a dice bag.

## Dice Bag Testing

First, we want to be able to specify the dice possibilities. In the game, we're going to have lots of min/max ranges, and so our dice should take that into account:

```



```

Second, we want to specify how many dice we're putting in the bag. The more dice, the more 'random' the situation can potentially feel. Too few dice, and the game will feel predictable.

