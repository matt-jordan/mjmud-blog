---
layout: post
title: "Combat Part 6: Bring Out Yer Dead"
---

Does it ever feel like programming is one long disappointing conversation you have with your computer where - deep down - you know you're to blame for everything?

Just me?

Okay.

_Sigh._

In this episode of "that was likely not the best idea", let's look at adding corpses to the game when a character surrenders its mortal coil.

## The Body Is Merely A Container

Today, most things that a character carries around or are dropped in a room are 'inanimates'. They have an underlying container that adds a bit of encapsulation around a Javascript array in `InanimateContainer`; this mostly serves to help find inanimates with case insensitive name searches and handle selecting an item that exists in a 'stack', e.g., "get 2.rat pelt from corpse". There's two things that count as an inanimate today: Armor and Weapons. Because this is JavaScript, we don't have to have a lot of complicated inheritance hierarchies: rather, there's an implicit interface that inanimates are expected to have. If we defined that contract, it would look something like this:

```
interface IInanimate {
	// Unique identifier
  get id() -> String;

  // Descriptive name of the item, generally how a player selects the item
  get name() -> String;

  // The item type that maps this to a data model
  get itemType() -> String;

  // Weight
  get weight() -> Number;

  // Short descriptive text, used when listed in inventories or rooms
  toShortText() -> String;

  // Save to the model
  async void save();

  // Load from the model
  async void load();
}
```

There's other properties that pertain more to "is this thing wearable", "can a player wear it", and "if this is a container, how do I interact with it." Each of those also sort of federate out into other 'shared interfaces' that are useful abstractions when thinking about basic properties that inanimate objects have.

The problem is, I only have Armor - which has the 'container' type properties - and Weapons. A corpse isn't a weapon. A corpse does contain things, which makes it sort of like Armor, but it isn't wearable [1], so making it a type of Armor would give it properties that it really shouldn't have.

What are some options?

1. Put it in Armor. I don't like this for the aforementioned reasons that wearing corpses is silly.

2. Make a base class, pulling out the inanimate bits, and have corpses be a new type of object. I'm not against this, and it's likely I'll do some extrapolation of common functionality into a base class of some sort. JavaScript has classes now, and I'm using plenty of them, but it really doesn't feel like it _wants_ a lot of inheritance.

3. Don't worry about base classes and just add a new object class of some sort, duplicating the methods you need. This is probably the right starting point. Once I've gotten things hammered out, I can see if things need to get refactored into a common base class.

## The Real Corpses Are The Friends We Made Along The Way

It's funny how the brain works. I've stared at the Armor class for awhile, pondering if I should just make a little corpse factory and generate them out of Armor instances. I didn't like it, and ended up freezing up for awhile things about it.

Writing out the options helped clear cobwebs - and, in many ways, trivialized the problem in front of me. A lot of small architectural issues are often like this: little problems that I tend to agonize over but that can easily be refactored if they don't work out.

And now I'm agonizing over what to name the damned class. Drat.

Forget it, I'm just calling it an inanimate.

```
const inanimateSchema = new Schema({
  name: { type: String, required: true },
  description: { type: String },
  isContainer: { type: Boolean, default: false },
  inanimateRefs: [{ type: inanimateRefSchema }],
  containerProperties: {
    weightReduction: { type: Number, default: 0 },
    weightCapacity: { type: Number, default: 10 },
  },
  weight: { type: Number, default: 1, required: true },
  durability: {
    current: { type: Number, default: 10 },
    base: { type: Number, default: 10 },
  },
});

```

Note that I used to call references to other inanimates (such as armor, or weapons) as `inanimateSchema`. I've now renamed it to `inanimateRefSchema`, which is also a bit more accurate. I'm going to go ahead and pull that out into another mongoose model module, as I'm referencing it all over the place. It's definitely time to do that as well, as the reference schema tells our load functions which Javascript class to use when instantiating the reference:

```
const inanimateRefSchema = new Schema({
  inanimateId: { type: ObjectId, required: true },
  inanimateType: { type: String, required: true, enum: ['weapon', 'armor'] },
});
```

Updating that enum in multiple files would be a bad idea.

The skeleton of the class can effectively follow what `Armor` has today:

```
class Inanimate {

  /**
   * Create a new inaninate object
   *
   * @param {InanimateModel} model - The mode underpinning this object
   */
  constructor(model) {
    this.model = model;
    this.durability = {
      current: 1,
      base: 1,
    };
    this.inanimates = new InanimateContainer();
    this._weight = 0;
    this.onWeightChangeCb = null;
  }

```

This already raises some interesting concerns: this is effectively a copy+paste of `Armor`. Now, I've already established that I don't want everything in `Armor` - particularly the notion of wearable locations or even class restrictions, but I'm definitely reproducing a lot of code. That means reproduced tests. Which I don't really want to write.

What I really want here is a Mixin - small bits of functionality that I can pull into the class. JavaScript doesn't have a good concept for this. I could use composition over inheritance - which is effectively what I'm doing with the `InanimateContainer` here. That may end up being the approach I take, but I do dislike the long chains of method calls that can result.

I know there's some thoughts floating out there on the internet on how to do some clever things with the `class` keyword in adding multiple Mixins to a class through `extends` on the fly. I may play around with that and see how I feel.

For now, I'm going to just run the risk of repeating myself a little bit and finish up this class, as it's mostly boilerplate.

## Corpse Factory Would Be A Great Metal Band Name

I'm running a little bit at risk, as I haven't written any tests yet.

**THIS IS BAD AND I KNOW IT**

At the same time, I'm at 99% confidence that I'm going to shred my classes apart in refactoring. Now, that means I really _should_ have tests to validate the behavior before and afterward. And I do, in `testArmor` and `testWeapon`. 

I'm going to come back here and read these paragraphs and see if my sense of foreboding was accurate.

The factor itself is pretty simple:

```
const corpseFactory = async (character) => {

  const model = new InanimateModel();
  model.name = `${character.toShortText()}'s corpse`;
  model.description = `The corpse of ${character.toShortText()}`;
  model.weight = character.weight;
  model.isContainer = true;
  model.containerProperties.weightReduction = 0;
  model.containerProperties.weightCapacity = 1000; // Just something large
  model.durability.current = Math.ceil(model.weight / 10);
  model.durability.base = Math.ceil(model.weight / 10);

  await model.save();

  const corpse = new Inanimate(model);
  return corpse;
};
```

Now I have a couple of options. I need to move everything a character is carrying and put it in the corpse. This includes all their wearable locations, as well as their inventory. I _could_ do that here, as we have the character and the corpse in the same place.

I think however I'm going to stick this into `Character`. Most of what I need to do is a manipulation of the character and not the corpse, which makes me feel like it belongs there. Plus, if I need to change things like how items are represented on the character, I'm more likely to catch the logic changes there than if I have to go into the factory method.

Plus, the factory method here is doing its job: make a corpse and returning it. The act of populating it is subtly different.

```
    const corpse = corpseFactory(this);
    if (corpse) {
      // Remove all equipment and put it in the corpse
      Character.physicalLocations.forEach((location) => {
        if (this.physicalLocations[location].item) {
          const item = this.physicalLocations[location].item;
          corpse.addItem(item);
          this.model.physicalLocations[location].item = null;
        }
      });

      // Move all hauled items into the corpse
      this.inanimates.all.forEach((item) => {
        corpse.addItem(item);
        this.inanimates.findAndRemoveItem(item.name);
      });
    }
    this.room.addItem(corpse);
```

As usual, every time I walk the physical locations I don't love it. This is sort of a combination of needing to have things indexable (dictionary) and yet wanting to walk it quite often since we have to do look-ups by item name (list). I may think about some better data structure for this in the future, as the code is kind of cumbersome and I find myself repeating it a lot.

## In Which I Discover I Was The Corpse All Along

Remember how I said not writing those tests was a bad idea?

_It was a bad idea._

After kicking off the server, I was largely able to kill a rat. And then the server crashed when saving.

What went wrong?

```
03:53:44.704Z ERROR spire-game: Failed to save room (roomId=61df835ae437a726fb9ef328)
  ValidationError: Room validation failed: inanimates.0.inanimateId: Path `inanimateId` is required., inanimates.0.inanimateType: Path `inanimateType` is required.
      at model.Document.invalidate (/Users/mjordan/Projects/Personal/spire-game2/node_modules/mongoose/lib/document.js:2923:32)
      at EmbeddedDocument.Subdocument.invalidate (/Users/mjordan/Projects/Personal/spire-game2/node_modules/mongoose/lib/types/subdocument.js:213:12)
      at /Users/mjordan/Projects/Personal/spire-game2/node_modules/mongoose/lib/document.js:2714:17
      at /Users/mjordan/Projects/Personal/spire-game2/node_modules/mongoose/lib/schematype.js:1280:9
      at processTicksAndRejections (node:internal/process/task_queues:78:11)

```

This then created a fun situation in which corpses were getting generated but characters were getting stranded as references in rooms. So that was a bit of great MongoDB CLI cleanup on top of having a crash to debug.

Not surprisingly, the bug is actually in my code above. First, I had forgotten to load the corpse object after instantiating it, which meant the `model` was set but none of the properties loaded in.

```
  await model.save();

  const corpse = new Inanimate(model);
  return corpse;
```

Should be:

```
  await model.save();

  const corpse = new Inanimate(model);
  await corpse.load(); // Oops

  return corpse;
```

Second - and this one at least is more subtle - Spawners keep track of the characters they've spawned so they know not to keep generating enemies infinitely:

```
    asyncForEach(mobsToGenerate, async (generator) => {
      const mob = await generator.generate();
      log.debug({ roomId: this.room.id, characterId: mob.id }, `Generated new ${mob.name}`);
      this.characters.push(mob);
    });
```

Unfortunately, the character never gets removed from the array on death, so the spawner thinks the characters is still wandering around. Rats.

This one is going to take some more work since the character will die somewhere not in the same room as the spawner. So we'll need an event that we fire so the spawner can be notified when the character croaks.

This isn't hard, but is yet another reason to write some darn tests.

Oh well. Live, learn, and repeat the same mistakes.

## Other Things to Refactor

1. There's now a lot of duplicate code in commonly named functions between `Inanimate`, `Armor`, and `Weapon`. I need to pull that out into either a 1/ base class or 2/ some more complicated scheme of Mixins.

2. I had a utility module called `inanimates` that contained the `loadInanimate` factory function along with the `InanimateContainer`. That all needs to get extracted into their own modules. I'm discovering I'm usually not a fan of modules that export multiple things. Sometimes it's appropriate, but I almost always want to just pull things out into their own modules eventually.

3. I didn't put corpses into the same module as `Inanimate`, and I'm glad. I need to extract out the rest of the factory functions.






[1] Unless you're Buffalo Bill, but let's not dwell on that.