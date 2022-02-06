---
layout: post
title: "Combat Part 2: Processing Combat Rounds"
---

After breaking down the requirements in the [last post](), I've started to write some unit tests for the Combat round. Right now, I'm thinking of having combat be represented by a pretty simple class that forms a relationship between an attacker and a defender, with a method that processes the combat on each tick.

Something like:

```
class Combat {
  constructor(attacker, defender) {
  	...
  }

  processRound() {
  	// Have the attacker take a swing at the defender and process what happens
  }
}
```

But before we write the implementation, let's try to write some tests first. I'm not always good at holding myself accountable to TDD, but when I do, I always end up with better code. So let's give that a shot.

## Test #1: If either party is dead, don't fight

This is pretty straight forward. We can imagine a scenario where as the combat manager is walking through combats, someone has killed one or either party in a combat. If that happens, we shouldn't process the combat and instead let the system know that someone is dead.

I'm not sure if I care a lot who died if both parties are dead - I think we'll catch that eventually - but for now there may be some different handling so I'll have different return results.

Javascript doesn't support enums, so I may need to tweak these 'return values' at some point.

```
  describe('if either combatant is dead', () => {
    describe ('if the attacker is dead', () => {
      beforeEach(() => {
        char1.modifiableAttributes.hitpoints.current = 0;
      });

      it('resolves the combat', () => {
        const uut = new Combat(char1, char2);
        const result = uut.processRound();
        assert(result);
        assert(result === Combat.RESULT.ATTACKER_DEAD);
      });
    });

    describe('if the defender is dead', () => {
      beforeEach(() => {
        char2.modifiableAttributes.hitpoints.current = 0;
      });

      it('resolves the combat', () => {
        const uut = new Combat(char1, char2);
        const result = uut.processRound();
        assert(result);
        assert(result === Combat.RESULT.DEFENDER_DEAD);
      });
    });
```

## Test #2: Attacker wins

As I've been pondering this test, it's pretty easy to see that this is a fairly opaque black box hiding the mechanics of the combat system. That's not a bad thing: the abstraction is supposed to hide the complexity. At the same time, combat has to be a bit random in a game. If it was deterministic, the game would be boring. So we have to have some way to 'rig the dice' in the game to test the potential outcomes. I'm not sure yet how I want to do that, but I'll put a stub in for it in this test's `beforeEach`. I'll likely alter it in some fashion later.

```
  describe('if the attacker damages the defender', () => {
    describe('if it kills the defender', () => {
      beforeEach(() => {
        char2.modifiableAttributes.hitpoints.current = 1;
      });

      it('resolves the combat', () => {
        const uut = new Combat(char1, char2);
        uut.setNextDiceRoll(20); // TODO: Likely to change in some fashion
        const result = uut.processRound();
        assert(result);
        assert(char2.modifiableAttributes.hitpoints.current === 0);
        assert(result === Combat.RESULT.DEFENDER_DEAD);
      });
    });

    describe('if it does not kill the defender', () => {
      beforeEach(() => {
        char2.modifiableAttributes.hitpoints.current = 100;
      });

      it('continues the combat', () => {
        const uut = new Combat(char1, char2);
        uut.setNextDiceRoll(20); // TODO: Likely to change in some fashion
        const result = uut.processRound();
        assert(result);
        assert(char2.modifiableAttributes.hitpoints.current !== 100);
        assert(result === Combat.RESULT.CONTINUE);
      });
    });
  });
```

## Test #3: Attacker whiffs

If the attacker swings and misses, we want to 1/ make sure that the defender wasn't altered, and 2/ that the combat continues.

```
  describe('if the attacker misses the defender', () => {
    it('continues the combat', () => {
      const uut = new Combat(char1, char2);
      uut.setNextDiceRoll(1); // TODO: Likely to change in some fashion
      const result = uut.processRound();
      assert(result);
      assert(char2.modifiableAttributes.hitpoints.current === 6);
      assert(result === Combat.RESULT.CONTINUE);
    });
  });
```

# Next Steps

I think there's enough here for me to write a pretty simple `Combat` implementation. I can likely start sketching out the `CombatManager` as well, but I think I'll write the implementation for `Combat` before thinking about how the higher level manager will orchestrate the combats that are happening.

I may digress slightly though and write a 'dice' implementation of some sort next. Being able to 'lean' on the dice roll is an important concept, as weighting dice is a game mechanic that can be pretty handy even outside of testing. I think I'll write some tests for that first, then come back and write the `Combat` implementation.
