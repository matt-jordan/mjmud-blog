---
layout: post
title: "Combat Part 5: There Is No Fighting In The War Room"
---

Now that we can have an attacker beat up a defender, we need to organize this madness. Let's make a `CombatManager`!

Or better yet, let's write some tests for it.

I know I'm going to need to test if a character is in combat or not. We'll have to restrict them from moving at a minimum, so I may as well have some method that provides that validation.

```
  describe('checkCombat', () => {
    describe('if the character is in combat', () => {
      it('returns true', () => {

      });
    });

    describe('if the character is not in combat', () => {
      it('returns false', () => {

      });
    });
  });
```

If a character can be in a combat being managed, then I have to have a way to add them to it. This one is a bit fuzzy for me - I do want characters to be in multiple combats, but not as the aggressor. That is: A can fight B, and B can fight C, and C can fight A, and D can fight A. But A can't also fight E.

Something like that.

```
describe('addCombat', () => {
    describe('when the attacker is already fighting', () => {
      it('returns false', () => {

      });
    });

    describe('when the attacker is not already fighting', () => {
      it('creates a new combat between two characters and returns true', () => {

      });
    });
  });
```

I suspect I'll come back to this with some simple off-nominal tests. Things like starting a new combat with a dead character, without a defender, etc. So I'll poke on this some more once I get the API a bit more fleshed out.

```
  describe('onTick', () => {
    describe('when a character dies', () => {
      describe('and they were defending', () => {
        it('removes any combats referencing that character', () => {

        });
      });

      describe('and they were attacking', () => {
        it('removes any combats referencing that character', () => {

        });
      });

      describe('and they were both attacking and defending', () => {
        it('removes any combats referencing that character', () => {

        });
      });
    });

    describe('when processing the combat round', () => {
      it('organizes the rounds by the attacker initiative', () => {

      });
    });

    describe('when the combat is inconclusive', () => {
      it('keeps the combat going', () => {

      });
    });
  });
});

```

This is pretty similar to the tests we wrote for `Combat`, which isn't surprising. The variant that we'll want to test is that it manages multiple characters. I may need to be explicit with that.

```
  describe('onTick', () => {
    describe('with A fighting B, B fighting C, C fighting A', () => {

    });
    describe('with A fighting B, B fighting A, C fighting A', () => {

    });
    describe('with A fighting B, B fighting A', () => {
```

## Sketching in the API

### checkCombat

I'm going to start with what should be the slightly easier APIs before I get to the interactions that happen in `onTick`. `checkCombat` is as good a place as any:

```
  describe('checkCombat', () => {
    describe('if the character is in combat', () => {
      it('returns true', () => {
        const uut = new CombatManager();
        uut.addCombat(char1, char2);
        assert(uut.checkCombat(char1) === true);
        assert(uut.checkCombat(char2) === true);
      });
    });
```

I never love it when a test could fail due to another method failing - in this case, `addCombat`. I could try and sneak around it and add the participants to an internal list directly, but I find it's better to be pragmatic about these types of dependencies and keep unit tests from knowing about the internal workings of a class as much as possible.

### addCombat

The tests for `addCombat` feel mostly reductive with what we have already, but that's okay. Repetition with minor variations is not the root of all evils.

```
    describe('when the attacker is already fighting', () => {
      it('returns false', () => {
        const uut = new CombatManager();
        assert(uut.addCombat(char1, char2) === true);
        assert(uut.addCombat(char1, char3) === false);
      });
    });

    describe('when the attacker is not already fighting', () => {
      it('creates a new combat between two characters and returns true', () => {
        const uut = new CombatManager();
        assert(uut.addCombat(char1, char2) === true);
      });
    });

```

### onTick #1

So, now we have a bit of a problem. Let's look at the first test:

```
    describe('with A fighting B, B fighting C, C fighting A', () => {
      describe('when a character dies', () => {
        describe('and they were defending', () => {
          it('removes any combats referencing that character', () => {

```

I'm missing two things here:
1. A way to see how many combats are happening. This isn't _strictly_ necessary, but I do want to check that we lose two of the combats - both the one where a character is attacking but also where they are being attacked.

2. We need a way to weight one of those combats, which means using `setNextDiceRoll` on the underlying combat object.

It would look something like this:

```
          it('removes any combats referencing that character', () => {
            const uut = new CombatManager();
            uut.addCombat(char1, char2);
            uut.getCombat(char1).setNextDiceRoll(20);
            uut.addCombat(char2, char3);
            uut.addCombat(char3, char1);
            uut.onTick();
            assert(uut.combats === 1);
          });

```

So I'll add `combats` property test, as well as a `getCombat` set of tests as well.

### combats

This is pretty straight forward, and doesn't need a lot of explanation. Test for 0, for 1, and for n (in this case, 2). I could do more, but this feels reasonable.

```
  describe('combats', () => {
    it('returns 0 if there are no combats', () => {
      const uut = new CombatManager();
      assert(uut.combats === 0);
    });

    it('returns the right number of combats', () => {
      const uut = new CombatManager();
      uut.addCombat(char1, char2);
      assert(uut.combats === 1);
      uut.addCombat(char2, char3);
      assert(uut.combats === 2);
    });
  });
```

### getCombat

We should get `null` if the combat doesn't exist, and we should get some valid object back if it doesn't. In this case, the attacking character should always be unique, so we can use that character as the parameter to the function. In order to avoid having to return a list of combats, I'll just define the method to only return the combat of the attacking character. If I change that behavior, I'll at least have a test that fails:

```
  describe('getCombat', () => {
    it('returns null if the character is not in combat', () => {
      const uut = new CombatManager();
      uut.addCombat(char1, char2);
      assert(uut.getCombat(char3) === null);
      assert(uut.getCombat(char2) === null);
    });

    it('returns the combat if the character is the attacker', () => {
      const uut = new CombatManager();
      uut.addCombat(char1, char2);
      assert(uut.getCombat(char1));
    });
  });

```

## Wrapping it up

Phew! This is at least a good start on the unit tests. I'm actually struggling a bit with the `onTick` tests - there's some behavior, like ordering combats by initiative, there are honestly tricky to test as a black box. Most folks will use things like sinon or mocks to determine the overall state, but I've always found that overusing mocks is a great way to have an incredibly brittle set of tests. So I'm not keen on testing the internal implementation details of the class.

I may change my mind if the logic ends up being complicated enough. If so, I'd feel more comfortable extracting out logic into a 'private' method on the class, then unit testing that method. Sure, that breaks the class encapsulation, but at least I won't have thirty references to `sinon` that I have to update when I change some part of the internal method signatures.

