Title: The openage Converter - Part I: Reading Data
Date: 2020-01-29
Tags: nyan, converter, assets
Authors: heinezen
Summary: We present the structure of the new converter and how we read data from .dat files.

In our [previous blogpost series]({filename}/blog/D0000-openage_mod_api_intro.md) about the openage modding API, we outlined how AoE-like gameplay mechanics will be implemented in openage and how they are modeled in our dedicated database language *nyan*. However, one issue with creating a new modding interface from the ground up is that we cannot directly use the data files from the original games. We have to convert them to our own logic and formats for them to work with the openage engine. This process is what this new blogpost series will be about.

Today's topic will be the overall structure of the new Python converter and its data reading module. There will be two more blogposts in the next week, covering the transformation from AoE2 structs to nyan API objects and the export to the file formats. We will mostly concern ourselves with *data conversion* and leave *media conversion* out for now as that part of the converter is less exciting to explore.

# Converter Workflow

![Structogram]({static}/images/C0000-converter-structure.svg)

Every game or game edition we convert will go through 3 different stages: ***Reader***, ***Processor*** and ***Exporter***.

In the *Reader*, we parse data or media files and put them into a structured format that we can use during conversion. The content read is passed into the *Processor* where the actual conversion takes place. Here, the data are first organized into conversion objects that resemble the original games structure (Genie-like objects). These are transitioned into objects that model the openage API (API-like objects) and then put into actual nyan API objects that can be exported to a file. The *Exporter* will take the converted objects and sort them into modpacks, while the very last stage of the conversion is printing the objects to nyan files and other human-readable formats along with requested media files.

Every stage is modular which means it can easily be exchanged or replaced. This comes in handy when we have to consider the multiple releases of AoE1 and AoE2. For example, the Definitive Edition of AoE1 will use an extended Processor stage compared to the original version of AoE1 due to the presence of new features and different mechanics. This means we can do a lot of code sharing and do not have to build a new converter for every edition of a game.

Note that from this point onwards we will exclusively talk about data conversion, or more specifically, converting gamedata from the `empires*.dat` files. Media conversion is a separate topic that we will leave out for now and maybe handle in an additional blogpost in the future.

## Reading and Storing

Our first goal in the conversion process is to access the data and load it into the converter in a usable format. Since AoE2's `.dat` file uses a binary format, we have to *serialize* the values before we can do anything with them. In this case, serialization means that we traverse the file byte by byte, extract the values and store them as Python structs. The format has been well documented over the years, so we know exactly at which offset what attribute can be found.

When we encounter an attribute value in the `.dat` file we have to worry about two things: How we **read** it and how we **store** it. The distinction between those two is a result of the different contraints values of the `.dat` file and values in our own database language `nyan` have. For example - depending on the attribute - integers in the `.dat` file can differ in:

* **Length**: An integer may be stored with length 8 Bit (`int8`), 16 Bit (`int16`) or 32 Bit (`int32`).
* **Signing**: Integers may have 1 prefix Bit to signal a negative value (`signed int`) or not (`unsigned int`).

Therefore, we have to specify the correct read type for every attribute in the `.dat`. Otherwise we would get the wrong values.

Additionally, even if two attributes have the same read type, we might want to use different storage types depending on the context the attribute is used for. The converter uses different storage types for attributes representing unit stats (e.g. HP, movement speed, damage vaues, ...) and resource IDs (e.g. sound or animation IDs). This becomes relevant when we want to compare two units later for the calculation of upgrades or if we want to determine the differences between vanilla AoE2 and a user made data mod.

![Read type and Storage type]({static}/images/C0000-read-vs-value-member.svg)

In our code, the serialization of the attributes looks like this:

```python
dataformat.extend([
    ...
    (READ, "hit_points", StorageType.INT_MEMBER, "int16_t"),
    (READ, "line_of_sight", StorageType.FLOAT_MEMBER, "float"),
    ...
    (READ, "dead_unit_id", StorageType.ID_MEMBER, "int16_t"),
    ...
    (IGNORE, "hidden_in_editor", StorageType.BOOLEAN_MEMBER, "int8_t"),
    (IGNORE, "old_portrait_icon_id", StorageType.ID_MEMBER, "int16_t"),
    (READ, "enabled", StorageType.BOOLEAN_MEMBER, "int8_t"),
    ...
    (READ, "resource_cost", StorageType.ARRAY_CONTAINER, SubdataMember(
        ref_type=ResourceCost,
        length=3,
    )),
    ...
])
```

For every attribute in the `.dat` file, the *Reader* module contains a 4-tuple that stores its' read mode, attribute name, storage type and read type.

**Read Mode**

This specifies whether we want to actually read the value (`READ`) or skip it (`IGNORE`). The latter read mode is used for values that are irrelevant to the conversion. Often they are leftovers from old beta versions (e.g. `old_portrait_icon_id`).

**Attribute Name**

A human-readable name for the attribute. This name will be used in the *Processor* stage to address them and read their values.

**Storage Type**

Here we tell the *Reader* which storage type it should use. Every type corresponds to a `ValueMember` subclass object that stores the attribute name, its value and its storage type. `ValueMember` also implements an auxiliary method for diffing.

**Read Type**

The read type of an attribute. For primitive types such as integers, strings or floats this will be a string, but it can also refer to other nested dataformat definitions like `ResourceCost` in the code sample above. This way, we only have to define the dataformat for a repeating struct once.

## Organizing

After reading the `.dat` file, we receive a full representation of its' content in an array of `ValueMember` instances. While the data we receive from the *Reader* already contains all the attributes and values we need later on, this output is too unstructured to be handled efficiently yet. If possible, we would like to operate on logical entities we know from the game like units, buildings, civs or terrains instead of cherry picking from a giant array of attributes.

![Data dump of GenieObjects]({static}/images/C0000-genie-objects.svg)

To do this, we transform the dumped values into converter objects that resemble the structure used in the original engine (hence *AoE-like* objects). These objects (`GenieObject`s) are very similar to what you would see in Advanced Genie Editor, just without the fancy GUI. For example, we put every `ValueMember` attribute belonging to an AoE2 unit into a `GenieUnit` object. The same process is repeated for every unit and every other logical structure that we can recognize in the game. Afterwards, each attribute will be assigned to an appropriate `GenieObject` subclass. Every subclass instance receives its' own list in a container which can be accessed in the *Processor* stage.

Not only is this object-oriented approach much easier to work with, we also no longer have to worry about where exactly logical entities (e.g. units) are located in the dump. Thus we can focus purely on its content and leave the legacy of the `.dat` file structure behind.

## To be continued...

That's it for now. We will continue next time when we transform data from the older AoE-like structure to the openage API.

Do you have more questions? Then let us know and discuss them with us and the community by visiting [our subreddit /r/openage](https://reddit.com/r/openage)!

As always, if you want to reach us directly in the dev chatroom:

* Matrix: `#sfttech:matrix.org`
* IRC: `#sfttech` on freenode.net
