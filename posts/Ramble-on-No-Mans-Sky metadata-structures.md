---
title: Ramblings on No Man's Sky's Metadata
date: 2023-12-3
permalink: /blog/2023/11/29/nms-metadata-ramblings/
---

For those who are new here, I spend an ungodly amount of my time working on stuff for No Man's Sky, there are plenty of links
around the place so feel free to look around. This article will be fairly short, mostly because it's only interesting to me but should be an interesting read if you're into this sort of stuff.

## What is the metadata?

There are quite a number of constants in the universe: the sky is blue, fire is hot and Hello Games employees go to work solely to write XML for 9 hours roughly each day. The No Man's Sky engine (which is called Skyscraper by the way, fancy that) uses XML documents encoded in a binary format for it's volatile structures and settings. The upsides are a bit questionable, it probably makes editing some stuff a bit less hands on in code, however it does not save on compile time as the structures are presumably compiled from XML to C++ source directly.

## What's neat about it?

All of the metadata's structure information is stored in the game's binary. If you're a source engine nerd it's a bit like those cool interface things all the cheaters are using, except probably a lot less useful since there are no vtables for accessing useful calls to the class itself. Each metadata object can be wrapped as a `cTkClassPointer`, which is a structure used mostly in during the process of the engine instancing game objects as scene nodes.

Just to cover it quickly, scene nodes in No Man's Sky are based on the Horde3D renderer's paradigm. Once they're created they're usually passed around with a lookup index. In NMS they've gone a bit further and modified most of the engine backend to use various 'Toolkit' classes, the lookup being a union called `TkHandle`.

Each of these metadata structures are eventually assigned to `cTkComponent`s and `cTkAttachments`, or if not there it's usually implemented somewhere in the `gApplication` flow which I will probably cover some other time because they're another interesting topic to make this website less empty. However, each structure has a number of utility calls for read/writing XML files and casting class pointers. Which aren't *terribly* useful however if you need to figure out a way to handle the game's XML formatting, referred internally as `MXML` (which is probably metadata xml), it's a bit useful. Despite this, for most people here you can use MBINCompiler (Which is the bread and butter of current No Man's Sky modding, check it out.) to create those files anyway, although they're EXML - which isn't MXML, which might not be what you need for your use-case.

What's way more interesting is the complex metadata handling they have behind the scenes to enable this, which is what I can cover now after getting the prologue for all of this out of the way.

Each metadata object is represented by a class known as `cTkMetaDataClass`, it looks something like this:

```cpp
class cTkMetaDataClass
{
  public:
    const char *mpacName;
    const uint64_t muNameHash;
    const uint64_t muTemplateHash;
    const cTkMetaDataMember *maMembers;
    const int miNumMembers;
};
```

The `muNameHash` is a very important thing to point out. HG loves hashes and if you want to resolve any of these classes at runtime you need their hash. They're a bit complicated using a combination of Murmur3 and SpookyHash2 to generate a recursive list of hashes based on each member which are all Murmur3'd (yes that's a word) near the end. What I do mostly to circumvent this insanity is hook `cTkMetaData::Register`, however it's difficult to get working on steam builds since it happens pretty close to startup.

Using this you can iterate over a list of `cTkMetaDataMember`, it's also a cool thing to point out they specifically don't have notation by default for most of these structures in XML so they generate them in at compile time.

Here's the structure for `cTkMetaDataMember`:

```cpp
class cTkMetaDataMember
{
  public:
    enum eType
    {
        ...
    };

    const char *mpacName;
    const unsigned int miNameHash;
    const char *mpacCategoryName;
    const unsigned int miCategoryHash;
    const cTkMetaDataMember::eType mType;
    const cTkMetaDataMember::eType mInnerType;
    const int miSize;
    const int miCount;
    const int miOffset;
    const cTkMetaDataClass *mpClassMetadata;
    const cTkMetaDataEnumLookup *mpEnumLookup;
    const int miNumEnumMembers;
    const FloatEditOptions mFloatEditOptions;
    const FloatLimits mFloatLimits;
};
```

`eType` contains a list of toolkit structures the game uses which I stripped for brevity. If you're curious you can always check out [renms](https://github.com/VITALISED/renms) for the all the classes you might be curious in seeing.

Anyway these are stored in a stack allocated vector somewhere around when the game runs. Not terribly important just worth mentioning.

## Interesting implementation specifics

The actual implementation for these classes are unions, it's actually interesting that Hello Games has a lot of their classes as unions despite the fact we no longer live in the 90s (we have the space for singletons to be an extra 8 bytes). Literally any class that has "manager" in the name seems to be a union of some sort, representing it's member capacity or a count of some sort. So, a metadata class generally has each metadata member and complete class object tagged alongside.

Another interesting design choice HG does to optimise space is compress the enums metadata uses to their smallest possible size. Enums in C++ are by default int32s but you can specify a smaller size, such as a uint8

Here's an example of an enum they use, generated by me.

```cpp
enum eGradient : uint8_t
{
    EGradient_None = 0,
    EGradient_Vertical = 1,
    EGradient_Horizontal = 2,
    EGradient_HorizontalBounce = 3,
    EGradient_Radial = 4,
    EGradient_Box = 5,
};
```

Whilst the implementation isn't really noteworthy, it's cool that they even bother doing this. Metadata is roughly the majority of structures the game uses so it sort of makes sense. (Also for anyone wondering, you can check the size based on `const int miSize` in `cTkMetaDataMember`.) 

## The end of the ramblings

If you like this sort of semi-tutorial/rambling style of writing, you probably can't let me know since there aren't any comments like a Youtube video; however feel free to connect with whatever means the site provides. Most of what I've been working on the last few months has been No Man's Sky related so you really should expect some more of this sort of stuff.

I wrote it Koutsie leave me alone now.
