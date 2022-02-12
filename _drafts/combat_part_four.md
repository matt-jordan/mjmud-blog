---
layout: post
title: "Combat Part 4: Back to the Pointy Sticks"
---

In [Part 2](), I had laid out some tests that fail. Because I love a good ditch of digression, in [Part 3]() I made a dice bag, because non-uniform distributions are bad. In this part, I'll implement the `Combat` class from Part 2 of Combat 101.

_Fair Warning: I actually haven't implemented this yet. I'm doing it as I write this post. I wonder how much backtracking will result._

## The Basics

First, let's get things in place so that the tests fail with actual test failures, and not with silly "no class or method found" exceptions (or the JavaScript equivalent). Just looking at our unit tests, we need at least the following:

```
class Combat {

  static get RESULT() {
    return {
      ATTACKER_DEAD: 1,
      DEFENDER_DEAD: 2,
      CONTINUE: 3,
    };
  }

  constructor(attacker, defender) {
    this.attacker = attacker;
    this.defender = defender;
  }

  processRound() {
    return Combat.RESULT.CONTINUE;
  }

}

```

Running the unit tests, we now get the following:

```
  1) Combat
       if either combatant is dead
         if the attacker is dead
           resolves the combat:

      AssertionError [ERR_ASSERTION]: The expression evaluated to a falsy value:

  func.apply(thisObj, args)

      + expected - actual

      -false
      +true
      
      at Decorator._callFunc (node_modules/empower-core/lib/decorator.js:110:20)
      at Decorator.concreteAssert (node_modules/empower-core/lib/decorator.js:103:17)
      at decoratedAssert (node_modules/empower-core/lib/decorate.js:51:30)
      at powerAssert (node_modules/empower-core/index.js:63:32)
      at Context.<anonymous> (file:///Users/mjordan/Projects/Personal/spire-game2/test/game/combat/testCombat.js:83:9)
      at processImmediate (node:internal/timers:464:21)
```

That test failure is due to us always returning `CONTINUE`, instead of what the unit test expects when the attacker is already dead: `ATTACKER_DEAD`. (Shocking return value, really)

Let's write that.

## The Easy Parts: Everyone Is Already Dead

It's possible that a combat gets invoked when a party is already dead. I'm not sure how that will happen, but if programming has taught me anything, it's that everything will always happen at some point in time. Disregarding bit flips, we can probably expect some spell to go off that nukes a character between `Combat` instances being processed, so we should just assume that one character is poking a corpse and handle that appropriately.

This is pretty trivial, and will let us get a few tests passing:

```
  processRound() {
    if (this.attacker.attributes.hitpoints.current <= 0) {
      return Combat.RESULT.ATTACKER_DEAD;
    }
    if (this.defender.attributes.hitpoints.current <= 0) {
      return Combat.RESULT.DEFENDER_DEAD;
    }

    return Combat.RESULT.CONTINUE;
  }

```

Woot! Two tests pass. On to the next problem.

```
  2 passing (250ms)
  1 failing

  1) Combat
       if the attacker damages the defender
         if it kills the defender
           resolves the combat:
     TypeError: uut.setNextDiceRoll is not a function
      at Context.<anonymous> (file:///Users/mjordan/Projects/Personal/spire-game2/test/game/combat/testCombat.js:109:13)
      at processImmediate (node:internal/timers:464:21)
```

## The Next Problem: God Plays Dice (And They Cheat)

The next problem is tricky. We want to test that if the attacking player damages and killed the defender, that we resolve the combat appropriately. But in a game filled with random numbers, that means we have to put our finger on the scale.

As the last test failure points out, we skipped over a stub for `setNextDiceRoll`. I'm conflicted about this, as it's pretty close to a function that turns our opaque testing to transparent testing. Maybe we'll find a reason to 'roll a natural 20' in the game at some point? It's quite possible.

I'll justify it to myself for now.

Let's set up some dice in the constructor:

```
    this.nextRoll = 0;
    this.diceBag = new DiceBag(1, 20, 8);
```

Implement the function that is missing:

```
  setNextDiceRoll(roll) {
    this.nextRoll = roll;
  }
```

And in our combat function, pick either `this.nextRoll` if it is non-zero or grab a dice roll if not:

```
    let roll;
    if (this.nextRoll > 0) {
      roll = this.nextRoll;
      this.nextRoll = 0;
    } else {
      roll = this.diceBag.getRoll();
    }

```

Eventually, attackers may make multiple attacks against defenders. I'm not sure how I want that to work yet, but it'll be pretty easy to add the above into a loop, so for now I'm just going to ignore it.

Now, we should just get the wrong result for the unit test:

```
2 passing (299ms)
  1 failing

  1) Combat
       if the attacker damages the defender
         if it kills the defender
           resolves the combat:

      AssertionError [ERR_ASSERTION]: The expression evaluated to a falsy value:

  func.apply(thisObj, args)

      + expected - actual

      -false
      +true
      
      at Decorator._callFunc (node_modules/empower-core/lib/decorator.js:110:20)
      at Decorator.concreteAssert (node_modules/empower-core/lib/decorator.js:103:17)
      at decoratedAssert (node_modules/empower-core/lib/decorate.js:51:30)
      at powerAssert (node_modules/empower-core/index.js:63:32)
      at Context.<anonymous> (file:///Users/mjordan/Projects/Personal/spire-game2/test/game/combat/testCombat.js:112:9)
      at processImmediate (node:internal/timers:464:21)

```

Progress!

## Poor Jud is Dead

Somewhere, I have a requirements doc that sketches out my thoughts for a combat system. (I should post those requirements somewhere other than in a `.md` file slapped in a directory someplace.) I'm mostly basing my MUD on the [D20 system](https://5e.d20srd.org/index.htm), with a nod to Dwarf Fortress in a few places and some tweaks to how armor works.

At a high level, combat works in three phases:
1. Pick a location. This uses a weighted percentage chance of which places the character can strike.
2. Roll to hit. Armor doesn't play a role here, but dexterity does, as do other skills.
3. Roll damage, applying damage reduction from armor.

I'm actually going to punt on Phase #1 for this post, and come back to it to make the game more interesting. We'll start with Phase #2: rolling to hit.

### Swing and a Miss

Phase #2 feels straight forward, as it involves character attributes that we have ready to use:

```
 Attacker Hit Role: D20 + Size Modifier + Attribute Modifier + Weapon Skill + Attack Bonus
 Defender Difficulty Check: 10 + Dexterity Bonus + Defense Skill + Dodge Bonus
 Did Player Hit = Attacker Hit Role >= Defender Difficulty Check
```

Okay, maybe it's not that straight forward. That's a lot of modifiers. Let's start with the most basic set up, where we take the dice roll and a stub for the attacker's hit bonus and compare it against the base defense score and the defender's defense bonus. We can flesh out these functions next.

```
    if (roll + this._calculateAttackerHitBonus() <= BASE_DEFENSE_SCORE + this._calculateDefenderDefenseBonus() ) {
      // Attacker missed
      return Combat.RESULT.CONTINUE;
    }
```

Calculating these bonuses is mostly work for another time. We don't have a lot of the stuff in place just yet to calculate weapon bonuses, and we haven't added skills or classes yet. I don't want to get too bogged down into doing all of these except for some simple attribute additions and size checks. We can start by simply adding the attacker's strength to their attack role and the defender's dexterity to the defense roll.

```
  _calculateAttackerHitBonus() {
    return this.attacker.getAttributeModifier('strength');
  }

  _calculateDefenderDefenseBonus() {
    return this.defender.getAttributeModifier('dexterity');
  }

```

We want small attackers to have an easier time at hitting larger characters. Likewise, larger enemies should have a penalty when swinging at things smaller than themselves. The size penalty can be calculated as:

```
 sizeBonus = defenderSize - attackerSize * 2
```

Where the sizes are given by the following enumeration:

```
 tiny = 0,
 small = 1,
 medium = 2,
 large = 3,
 giant = 4,
 collosal = 5,
```

So a `tiny` creature striking at a `collosal` creature will get a +10 bonus to hit, while the reverse relationship gets a -10. I'm already storing the size of the creature on the character model, so I'm just going to add a simple dictionary that performs this conversion for us. For now, I'll just add this to the `Combat` class, as the numeric conversions just make sense in the context of fighting. If that changes, I can move it to `Character`.

```
const sizeToNumber = {
  tiny: 0,
  small: 1,
  medium: 2,
  large: 3,
  giant: 4,
  collosal: 5,
};

...

  _calculateAttackerHitBonus() {
    const sizeBonus = (sizeToNumber[this.defender.size] - sizeToNumber[this.attacker.size]) * 2;
    const attributeBonus = this.attacker.getAttributeModifier('strength');

    return sizeBonus + attributeBonus;
  }
```

Okay, so now we're swinging. Let's do some damage.

### I Remember Damage

Usually, characters will do damage by using a weapon. However, that's another complicated thing to implement: what if characters change weapons during combat? How do different weapon types impact the combat? I want to keep this simpler, so I'm just going to assume that the characters are unarmed.

If a character is unarmed, we assume they're swinging their fists (or paws) for now. A default attack would be a good idea to add to each of our 'sub-classes' of different characters, but that's another improvement for another day. (So many things to tweak!)

```
  _calculateAttackerDamage() {
    const strengthModifier = this.attacker.getAttributeModifier('strength');
    const min = Math.max(0, strengthModifier);
    const max = Math.max(1, strengthModifier + 1);

    return getRandomInteger(min, max);
  }

  processRound() {
    ...
    // After the hit roll

    const damage = this._calculateAttackerDamage();

  }
```

There's a couple of things I don't love about this just yet, but I'm going to live with it. First, I'm using random integer generation as opposed to a dice bag. I do want to use a dice bag at some point, but damage ranges are likely to change as weapons get changed out, and I want to think a bit about how to handle that. Second, we'll have to refactor how we're applying the Strength bonus. Strength bonuses go in both directions: it can add damage if you're strong, but reduce damage if you're weak. However, we shouldn't ever take the max damage that a player can do below 1, and the minimum below 0. (Negative would be healing...)

So we'll re-visit this later, but it's some damage from a punch. Let's apply that damage.

```
    const damage = this._calculateAttackerDamage();
    const delta = this.defender.attributes.hitpoints.current - damage;
    this.defender.attributes.hitpoints.current = Math.max(delta, 0);

    if (this.defender.attributes.hitpoints.current === 0) {
      return Combat.RESULT.DEFENDER_DEAD;
    }

    return Combat.RESULT.CONTINUE;

```

How'd we do?

```
  2 passing (296ms)
  1 failing

  1) Combat
       if the attacker damages the defender
         if it kills the defender
           resolves the combat:

      AssertionError [ERR_ASSERTION]: The expression evaluated to a falsy value:

  func.apply(thisObj, args)

      + expected - actual

      -false
      +true
      
      at Decorator._callFunc (node_modules/empower-core/lib/decorator.js:110:20)
      at Decorator.concreteAssert (node_modules/empower-core/lib/decorator.js:103:17)
      at decoratedAssert (node_modules/empower-core/lib/decorate.js:51:30)
      at powerAssert (node_modules/empower-core/index.js:63:32)
      at Context.<anonymous> (file:///Users/mjordan/Projects/Personal/spire-game2/test/game/combat/testCombat.js:112:9)
      at processImmediate (node:internal/timers:464:21)
```

Drat. Debugging time!

## You Roll a 1 for Implementation

Our assertion is on the following line in the test:

```
        assert(char2.attributes.hitpoints.current === 0);

```

Which means we didn't actually do any damage to the defending character. Let's see if we swung and hit:

```
    console.log(roll + this._calculateAttackerHitBonus());
    console.log(BASE_DEFENSE_SCORE + this._calculateDefenderDefenseBonus());
    if (roll + this._calculateAttackerHitBonus() <= BASE_DEFENSE_SCORE + this._calculateDefenderDefenseBonus()) {
      // TODO: Add a message that you missed
      return Combat.RESULT.CONTINUE;
    }
```

Results in:

```
20
10
        1) resolves the combat

```

So yup, we're hitting. Not surprising as I was pretty sure I didn't mess up the attack/defense bonuses with the test characters, but definitely worth checking.

_Side Note: Yes. console.log. Never underestimate some well placed printfs._

Let's check damage:

```
    const damage = this._calculateAttackerDamage();
    const delta = this.defender.attributes.hitpoints.current - damage;
    console.log(`Damage: ${damage} / Delta: ${delta}`);
    this.defender.attributes.hitpoints.current = Math.max(delta, 0);

```

Results in:

```
Damage: 0 / Delta: 1
        1) resolves the combat

```

Okay, so we're not generating any damage. Which, as I type this, makes sense. Remember, we roll a dice from 0 to 1 inclusive and add the Strength bonus of the attacker. Which means that there's a very good chance that if the Strength bonus is 0, the attacker will do no damage and we won't kill the defender.

*SIGH*

Let's bump up their strength:

```
    char1Model.race = 'animal';
    char1Model.attributes = {
      strength: { base: 18, },
      dexterity: { base: 10 },

```

And we get:

```
  5 passing (350ms)

```

Huzzah!

# Wrap-Up

What did I learn?

I continue to be happier with the code that I write when I write the tests first. It's funny because in the moment, it can be frustrating: it sometimes feels so... slow. Particularly when I'm backtracking in the implementation, or when I've got an implementation rolling forward and I realize that I've missed something or I'm deviating from the test definition. But the reality is that the code that I write when I'm in that state of mind is almost uniformly worse code than when I'm going slowly, deliberately, writing just enough to get the next failing test to pass.

I did that here, although I'm a bit worried by how many untestable calculations I'll likely end up adding into this class. The good news is that I _can_ test the private methods I'm adding, which may be worth doing as the calculations get more complicated. Something to ponder for when I have to add skills, different weapon types, and more.
