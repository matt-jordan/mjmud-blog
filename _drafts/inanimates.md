---
layout: post
title: "Creating a MUD"
---

# Inanimate Leakage

I'm not happy with how I'm handling what I call "inanimates" in the game. At its basic, an inanimate is something that a character interacts with. Think something that you pick up, look at, hold, wear, etc. My assumption was that having a basic 'type' for most things in the world would give a lot of freedom/flexibility in defining those things, and since I'm using a dynamically typed language, I don't really have to worry too much about having some obscene inheritance hierarchy nonsense. While this is probably true, it does create a bit of complexity that is bothering me.

As an example, let's look at how a player currently has to wear a piece of armor or wield a weapon. Both are triggered by the `Wear` command. When that command is processed, we find the inanimate object the player specified - which in this case, has to be in the `inanimates` list on the character - find the location that player specified or one that is empty and that supports the item in question - and then move the item to that location.

First, we see if the location specified by the player makes any sense:

https://github.com/matt-jordan/mud-backend/blob/main/src/game/commands/default/WearItem.js#L43
```
    let location;
    if (this.location) {
      location = textToPhysicalLocation(this.location);
      if (!location) {
        character.sendImmediate(`${this.location} is not a place on your body`);
        return;
      }

      if (character.physicalLocations[location].item) {
        character.sendImmediate(`You are already wearing something on your ${this.location}`);
        return;
      }
    }
```

I already don't like that second check to see if `character` is wearing an `item` on their person already. This leak from the `character` class is the first thing I want to clean up, as it's easy to forget to check if `item` is `null` or not.

If the player specific a location, we then try to find the `item` they specified:

https://github.com/matt-jordan/mud-backend/blob/main/src/game/commands/default/WearItem.js#L56
```
    const item = character.inanimates.find(item => item.name === this.target);
    if (!item) {
      character.sendImmediate(`You are not carrying ${this.target}`);
      return;
    }
```

This definitely isn't going to work long-term. We're doing an explicit check `String` comparison for the name of the item in the list of `inanimates` the `character` is hauling around. But that isn't the experience we want as the player: if the name of my weapon is "Longsword of the Bear", I don't want to have to type that every time I look at it, I want to type "longsword" at most.

It's going to get even more complicated when we have to add checks for character levels, physical attributes, skill levels, and more. Yikes.

This mess isn't constrained to `WearItem`. We can see this is equally in `RemoveItem`. Here in the code I'm trying to handle a character saying 'remove ring', where 1/ the ring could be on either finger or 2/ ring could be ambiguous and there are multiple rings on the character.

https://github.com/matt-jordan/mud-backend/blob/main/src/game/commands/default/RemoveItem.js#L60
```
      const candidates = [];
      PlayerCharacter.physicalLocations.forEach((location) => {
        if (character.physicalLocations[location].item
          && character.physicalLocations[location].item.name === this.target) {
          candidates.push({
            item: character.physicalLocations[location].item,
            location,
          });
        }
      });
      if (candidates.length === 0) {
        character.sendImmediate(`You are not wearing ${this.target}.`);
        return;
      }
      if (candidates.length > 1) {
        character.sendImmediate(`Which ${this.target} do you want to remove?`);
        return;
      }
      character.physicalLocations[candidates[0].location].item = null;
      item = candidates[0].item;
      location = candidates[0].location;
```

That iterating over the potential locations bothers me in a command. What if we want to support an 8-armed beast each wielding a sword, and the player successfully disarms one of them? Not only will the physical locations be different, but having the logic for finding and removing an item live in a player command action is clearly not the best place for it. We'll end up with duplicate logic for manipulating items on characters, which is likely to lead to not only a lot of repetitive code, but bugs and painful refactoring.

# Thoughts on Doing it Better

There's at least two distinct problems here:

1. Javascript arrays have too much surface area for handling inanimates across Characters, Rooms, and Armor types that act as a container (backpacks, belts, etc.). We can scope out a replacement that supports a minimal set of operations and that wraps up a Javascript array.

2. Character physical locations need to be wrapped up and protected by some facade. This would let us hide all the logic checks that we want to add over time.

Let's see what this might look like.

## Inanimate Container

Looking at the existing way we use the array in the existing code, there's three broad operations we perform that have side effects or complicated logic:
1. Add items
2. Find and remove an item
3. Check to see if an item exists

The third one is likely more a function of needing to see if an item exists before removing it, so we'll focus on the first two. I'm also going to go with a 'loose' wrapper around an underlying array, so I don't have to reproduce every function or attributes, like `length`.

Writing some tests first:

```
  describe('addItem', () => {
    const uut = new InanimateContainer();
    uut.addItem(item);
    assert(uut.inanimates.length === 1);
  });
```

This really can't fail. We also know that adding items to a container can cause weight adjustments, as we see in the `PlayerCharacter` class:

```
  addHauledItem(item) {
    this.carryWeight += item.weight;
    item.onWeightChangeCb = (item, oldWeight, newWeight) => {
      this.carryWeight -= oldWeight;
      this.carryWeight += newWeight;
    };
    this.inanimates.push(item);
  }
```

But right now I'm not sure what to do with that. The callback is on `Armor` currently.









## asdf



