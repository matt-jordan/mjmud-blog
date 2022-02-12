---
layout: post
title: "Combat Part 3: A Bag of Dice"
---

Many years ago, I read a great post on how random numbers are bad for games, particularly role-playing games. I can't find it, but if I ever do, I'll come back and link it here because it did a great job of explaining the concept.

I'll try to summarize.

There's at least some [statements floating around](https://github.com/ckknight/random-js) that the implementation we have for picking between two integer values in [randomInteger]() will not create a uniform distribution. For where I'm using that right now - picking an exit in a room, deciding if a monster will wander - that's probably not that big of a deal. When determining combat results however, you really don't want to bias towards hitting too often or missing too often. So that's one consideration.

Another consideration is simply that truly random is *bad* for games. No one wants to roll a '1' 20 times in a row, but that is perfectly possible with a large enough sample size. We want to limit the number of potential bad and good rolls that can happen, while still keeping things 1/ uniformly distributed and 2/ semi-random.

Consider, instead, a dice bag. For our purposes, we'll say we're rolling six sided dice, and we have 36 dice. With a uniform distribution, we'd expect six instances of each value: 6 1's, 6 2's, etc. We then put all the dice in a bag and shake it up.

While it's still possible to pull 6 1's in a row, once we've exhausted them out of the bag we're done with the 1's. We can't roll a critical failure again until we've exhausted the other 30 dice. That gives the character some reprieve from a non-infinite-but-very-long string of bad luck.

So: let's take a digression from making virtual characters kick each other, and instead let's make a dice bag.

## Dice Bag Testing

First, we want to be able to specify the dice possibilities. In the game, we're going to have lots of min/max ranges, and so our dice should take that into account:

```
  it('generates the expected values for a single set', () => {
    const bag = new DiceBag(1, 6, 1);
    const results = [];
    for (let i = 0; i < 6; i += 1) {
      results.push(bag.getRoll());
    }
    results.sort();
    for (let i = 0; i < 6; i += 1) {
      assert(results[i] === i + 1);
    }
  });

```

Second, we want to specify how many dice we're putting in the bag. The more dice, the more 'random' the situation can potentially feel. Too few dice, and the game will feel predictable.

```
  it('generates the expected values for multiple sets', () => {
    const bag = new DiceBag(1, 4, 4);
    const results = [];
    for (let i = 0; i < 16; i += 1) {
      results.push(bag.getRoll());
    }
    results.sort();
    for (let i = 0; i < 4; i += 1) {
      for (let j = 0; j < 4; j += 1) {
        assert(results[i * 4 + j] === i + 1);
      }
    }
  });
```

I'm not sure how I'd test to see if the values coming out are actually random. I think I'll let that be a small-ish gap in my testing for now.

Finally, I want the bag to appear to be infinite. As we exhaust a set of values, we should make a new set with a new distribution, and allow the consumer of the bag to keep drawing from them.

```
  it('generates expected sets when we exhaust the bag', () => {
    const bag = new DiceBag(1, 8, 1);
    const results = [];
    for (let i = 0; i < 16; i += 1) {
      results.push(bag.getRoll());
    }
    results.sort();
    for (let i = 0; i < 8; i += 1) {
      for (let j = 0; j < 2; j += 1) {
        assert(results[i * 2 + j] === i + 1);
      }
    }
  });
```

## Bag O Dice

Sketching out the contract from my tests:

```
/**
 * @module lib/DiceBag
 */

/**
 * A class that produces dice rolls in something slightly better than pseudorandom
 * but still not fully predictable ways
 */
class DiceBag {

  /**
   * Create a new dice bag
   *
   * @param {Number} min  - The minimum roll a dice can have
   * @param {Number} max  - The maximum roll a dice can have
   * @param {Number} sets - The number of sets of dice to put in the bag
   */
  constructor(min, max, sets) {
    this.min = min;
    this.max = max;
    this.sets = sets;
  }

  /**
   * Pull a result from the bag
   *
   * @return {Number}
   */
  getRoll() {

  }
}

export default DiceBag;

```

Let's start by storing the sets in an `Array<Number>` that isn't shuffled:

```
  constructor(min, max, sets) {

  ...

    this._dice = [];
    for (let i = 0; i < this.sets; i += 1) {
      for (let j = min; j <= max; j += 1) {
        this._dice.push(j);
      }
    }
  }
```

Now we need to shuffle the array. I'm not going to do anything too fancy for this; something like the following should suffice (in bad pseudocode):

```
  for 1 to n:
    r = rand(0 to Array.length)
    val = Array.slice(r, 1)
    new_array.push(val)
  return new_array
```

I can already anticipate I'm going to do this again when I exhaust the `_dice` list. That actually means I'm better off just having two lists of values: one that is an 'exhausted' list of dice and the other that is the shuffled list of dice. Yes, that's a trade-off in storage space (`2n`), but this is Javascript, not C, so I'm not going to worry about that.

That changes what I had in the constructor to the following:

```
    this._dice = [];
    this._exhausted = [];
    for (let i = 0; i < this.sets; i += 1) {
      for (let j = min; j <= max; j += 1) {
        this._exhausted.push(j);
      }
    }

    this._shuffle();

```

Next up is `_shuffle`, which should randomly pluck dice from our exhausted pool and put it into the active dice pool. `_shuffle` ends up being a small 'private' method that looks something like the following:

```
  _shuffle() {
    do {
      const index = randomInteger(0, this._exhausted.length - 1);
      const element = this._exhausted.splice(index, 1);
      this._dice.push(...element);
    } while (this._exhausted.length > 0);
  }
```

Nothing like a good old fashioned `do`/`while` loop to make you feel nostalgic.

Now that we can shuffle the dice bag on creation or when the dice are exhausted, we can write the `getRoll` function:

```
  getRoll() {
    if (this._dice.length === 0) {
      this._shuffle();
    }
    const roll = this._dice.splice(0, 1);
    this._exhausted.push(...roll);
    return roll[0];
  }
```

And tada! Our tests all pass. One cookie for the developer.

```
  DiceBag
    ✔ generates the expected values for a single set
    ✔ generates the expected values for multiple sets
    ✔ generates expected sets when we exhaust the bag
```

