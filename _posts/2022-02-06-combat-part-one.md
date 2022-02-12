---
layout: post
title: "Combat Part 1: Requirements"
---

_West Seattle. It was cloudy-ish. Then sunny unexpectedly._

# Pointy Sticks

I've been excited to start writing the combat system for the game for some time now. It takes quite a bit to get to this point: rooms, characters, movement, commands, event loops, and more. So now that I've got most of that in place, let's give folks in the game the opportunity to beat something up.

_Witness the violence inherent in the system_

I've been mulling over how I want to structure this, and I've come up with a loose set of functional requirements to help guide the design. Note that I'm not putting a _lot_ of thought into this, but enough to shape how I think I'll tackle the coding.

1. Combat occurs within a room.
2. A character cannot attack more than one other character at a time, unless it's via some special attack mechanism.
3. A character can attack another character that is attacking (yet again) a third character.
4. Combats are resolved on the main game loop, outside of any initial attacks that occur from a player initiating combat with their character.
5. Characters who die don't get to make their attacks.
6. Characters can switch the focus of their attacks.
7. Characters cannot leave a room while being attacked.
8. If a character is attacked and they are not attacking anyone, they start attacking their attacker.

Let's extrapolate what these requirements likely mean.

## Requirement #1: Combat occurs within a room

MUDs are generally room driven, and mine is no exception (see [room.js](https://github.com/matt-jordan/mud-backend/blob/main/src/game/world/room.js)). Most interaction happens in these self contained bubbles, which limits how we have to think about physics: you don't have to worry about blast radius when it's all happening in a single cartesian point. Deciding up front that I'm going to mostly stick with this model makes the problem a lot simpler as I don't have to think about cross-room combat. While I could conceive of a situation where a player has their character use a ranged weapon or a spell to nuke something they can't see, that makes this way more complicated and messy than I want right now. Maybe I'll come back to that idea some day.

If we ever want that feature, we could create some special mechanism that 'draws' characters hit by another character outside of their room to the room with the offending character, then insert them into the normal combat mechanics. But I don't want to over-index on that idea out of the gate.

## Requirement #2: One character at a time

This requirement sets up at first a bi-directional relationship between two characters, although as we'll see with requirement #3, it's not that simple. But it does mean that as we process combats, we shouldn't generally have a situation where characters are getting to attack multiple times.

On the other hand, I do think we need a higher level concept outside of this relationship to manage the combat. We need to have some way to answer the question "who all is attacking this character" so we can implement some mechanisms like a special attack that attacks each of your attackers once. So a simple bi-directional relationship won't cut it.

## Requirement #3: It's a directed graph

With cycles, no less. Luckily we shouldn't have to walk it.

You can have Character #1 attacking Character #2, who is attacking Character #3, who is attacking Character #1. This starts to frame up the objects and how we keep track of things. Each Character is the 'owner' of some combat against another character, with some form of manager keeping track of all of these combats that are occurring within a room. Likewise, that combat does need to know who the recipient of the combat is.

In my head, I'm thinking we need a `Combat` function or class and a `CombatMangaer` function or class.

## Requirement #4: How combat is started and processed

Effectively, this tells me we should do most of the updates in the `onTick` handlers, which I expected. However, we do allow players one advantage over the game controlled characters in that they get to start combats with a 'free round' of attacks before the ticks start processing. I actually like this concept: it rewards active playing, and gives some character classes like the rogue an opportunity to do some big damage up front.

## Requirement #5: Death happens

If you die, you don't get to hit back. This opens up a fun bit in thinking through how we manage the ordering of combats - which is another reason for a `CombatManager` of some sort, likely owned by the `Room`. Rather than walking through combats in the same order each time, we can have a `CombatRound` round get constructed from the `Combat`s by the `CombatManager` that orders things based on some 'initiative' skill or concept. There's a big bonus to swinging before your enemy, particularly when combat gets tight between two evenly matched participants.

I'm not sold that I need a `CombatRound`, that starts to feel a bit Java-y to me. But I'll think about it some more.

The fact that we're introducing death means we'll need a whole host of concepts: corpses, an `onDeath` handler to update various other parts of the game that this character is now out of action, a way to stop the player from interacting with their dead player, and more. There's a lot in there to unpack that I'll have to think about and will likely create a whole series of patches.

## Requirement #6: Attacks change

Giving players the ability to choose who to focus their attacks on gives them an element of strategy when getting attacked by multiple enemies. So we'll allow someone to effectively remove a combat they own and create a new one at any time.

## Requirement #7: No running away

The `Move` command - or where it is processed for characters in `Character.moveToRoom` - should be disabled during combat. We'll eventually want to make a 'flee' command or give enemies the opportunity to flee when their health gets low. Fleeing opens up all sorts of potential game changes, particularly once we introduce the notion of having a party. I'll probably hold off on this mechanism for a little bit until I'm pretty happy with the structure of the combat system. In the meantime, we'll just make it so that once you're in a combat, you're stuck there.

## Requirement #8: Auto-attack

In general, this makes sense, as we don't let you run away anyway once something takes a swing at you. Likewise, we want enemy character controlled by the game to start attacking once you attack them. We may eventually want to have something not auto-attack, particularly once we think about adding a concept of 'aggression', or how we'll track which characters in a party monsters should prioritize.

Next time: the initial structure of the combat system.


