# SHARD Format: Version 0

This document defines the data format of the [SHARD (Shard Highly Augmented Region Data) format][shard-format] in
version 0. It contains the expected data fields and encodings that are necessary to serialize to and deserialize from
SHARD files into their in-memory components. Libraries that want to use SHARDs need to adhere to this specification.
When in doubt, the reference implementation may be consulted.

## Disclaimer

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in [RFC 2119][rfc-2119].

## Definitions

Below is a list of terms and explanations that are either specific to Minecraft, the SHARD format, or both. Those
explanations should not be taken for technical implementations, but rather as an orientation regarding the meaning of
individual concepts within Minecraft or the format. Therefore, if more information is needed on those components,
appropriate external resources may contain more comprehensive explanations on these terms and their formats.

### UUID

[UUIDs (Universally Unique Identifiers)][uuid-wiki] (also known as GUID \[Globally Unique Identifier\]) are unique
labels that can be used to unambiguously identify objects/entities. A UUID has entropy of 128 bit, and while the
chance for collisions (the same UUID generated twice) is no zero, it is negligible for practical use cases. Therefore,
UUIDs can be assumed to be unique. In Minecraft, they are used to identify players, entities and other information.

### NBT

[Named Binary Tag (NBT)][nbt-wiki] is the primary binary format used by Minecraft to store various information on
Minecraft worlds and their inhabitants. It contains specialized types for most primitive and frequently used complex
data types that can be serialized to binary data. We use NBT to store information on entities and block entities.

### SNBT

[Stringified Named Binary Tag (SNBT)][snbt-wiki], which is also known as *data tag*, is the simplified form of NBT that
is represented as a string. This representation is often used in commands and consists of key-value pairs in a format
that resembles JSON. This format is used to represent complex information on entities as a single string.

### Resource Location

[Resource Locations][resource-location-wiki] or *resource identifiers* are namespaced keys/references that specify the
logical location of a specific resource within one of Minecraft's registries. Those keys consist of the namespace
(`minecraft` or another vendor like `justchunks`) and the name of the resource itself (for example `stone`). Those parts
are then joined together, separated by a colon: `minecraft:stone`. For Minecraft's resources, the namespace can be
omitted, as this is the implicit default value: `stone` is therefore equal to `minecraft:stone`.

### Block State

[Block States][block-state-wiki] represent instance of a specific block in the SHARD. This includes information like the
general block type (material) and its specific properties. The properties define for example how that block is oriented,
if it is activated, waterlogged, burning or which face of the block is attached. Two blocks at different locations that
are otherwise identical have the same block state.

### Block Entity

[Block Entities][block-entity-wiki] are special instances of blocks that hold additional information on top of normal
[Block States](#block-state). Only specific block types have corresponding entities. Those entities hold complex (and
often dynamic) data that often interacts with the world around it. As a result, their data is stored separately, as it
would be too complex (or volatile) to store as a [Block State](#block-state). Typical examples are chests, furnaces and
beacons that are represented as Block Entities, where their data (inventory, settings) is specific to each position they
may exist. Formerly, Block Entities were referred to as *Tile Entities*.

### Entity Type

[Entity Types][entity-type-wiki] are the distinct types the individual entities belong to. They are represented by
[Resource Locations](#resource-location) and control which NBT data set is valid for an entity. For example, if an entity
is of type `minecraft:bee`, there are many additional data fields in the NBT that store information on nectar, the
position of the hive or whether the bee has lost its sting.

### Entity

[Entities][entity-wiki] are single instances of a specific [Entity Type](#entity-type) at a specific position relative
to the origin of the SHARD. Entities are identified by a [UUID](#uuid) and contain dynamic data, that is based on the
[Entity Type](#entity-type). SHARDs don't contain the UUID of entities, so that the platform can assign a new identifier
according to the rules of this implementation. Most platforms will just generate a random new UUID for each entity. The
format for all entities can be read in the [Minecraft wiki][entity-format-wiki].

### Biome

[Biomes][biome-wiki] are specific environmental aspects of regions within Minecraft worlds. They affect the rendering
and the world mechanics like the formation of snow, whether it will rain and the spawn chance of certain mobs. Biomes
are independent of blocks and biomes can be different along each of the three axes. They are identified by their
[Resource Location][resource-location-wiki] in a dynamic registry.

### Config Position

Config Positions are JustChunk's configuration embed format. They are generated from text display entities and contain
configuration associated with this location. This could, for example, mean some kind of NPC that should be placed there
while loading the world. When exporting a world, all Config Positions are scanned and extracted, so they don't show up
as normal entities but instead become part of the world configuration.

## Endianness

All types and specifications are expected to be serialized and deserialized in Big Endian [Byte Order][byteorder-wiki],
unless otherwise specified. That means the most significant byte comes first and is stored at the smallest address of
the resulting file/stream.

## Data Types

These are the simple (primitive) data types that are used within the specification. More complex types are included
within the specification itself. The encoding and meaning of these types is strictly defined, and therefore, they are
not part of the specification.

| Data Type | Bytes | Description                                                        | Notes                                 |
|-----------|-------|--------------------------------------------------------------------|---------------------------------------|
| Byte      | 1     | Signed byte with range -128 to 127                                 |                                       |
| Short     | 2     | Signed short with range -32768 to 32767                            |                                       |
| Int       | 4     | Signed integer with range -2147483648 to 2147483647                |                                       |
| Long      | 8     | Signed long with range -9.223372e+18 to 9.223372e+18               |                                       |
| UByte     | 1     | Unsigned byte with the range 0 to 255                              |                                       |
| UShort    | 2     | Unsigned short with the range 0 to 65535                           |                                       |
| UInt      | 4     | Unsigned integer with the range 0 to 4294967295                    |                                       |
| ULong     | 8     | Unsigned long with the range 0 to 1.8446744e+19                    |                                       |
| Float     | 4     | Signed float in IEEE 754 floating-point "single format" bit layout | Preserves NaN (Not a Number) values   |
| Boolean   | 1     | Boolean value that is either 0 or 1                                | Only the least significant bit is set |

## Specification

A single SHARD file must include exactly one [SHARD](#shard). No data is allowed before or after this SHARD and the
magic bytes have to be placed at the very start of the file in order for MIME type detection to work. If used within
data streams, additional SHARDs may follow, but the user has to make sure that the next magic bytes are right before
the read cursor when trying to read the next SHARD from that stream.

It is guaranteed that only one SHARD will be read and the read cursor will remain right after the last byte that
belonged to the SHARD that was read.

### SHARD

A root object for the SHARD file format. This object represents the entire SHARD with all its stored data. A file
should only contain a single SHARD definition.

| Name                   | Type                                   | Description                                                        | Notes                                                                                                                                                   |
|------------------------|----------------------------------------|--------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Magic Bytes            | Byte[]                                 | Magic Bytes to perform MIME type recognition/integrity check       | Always `SHARD FILE FORMAT` + `0x00` encoded in UTF-8 <br/>(`0x53 0x48 0x41 0x52 0x44 0x20 0x46 0x49 0x4C 0x45 0x20 0x46 0x4F 0x52 0x4D 0x41 0x54 0x00`) |
| Version                | UByte                                  | Version number to indicate the appropriate codec                   | Always `0` for this version of the specification                                                                                                        |
| Unique Identifier      | [UUID](#uuid-1)                        | Identifier to unambiguously reference this SHARD                   | Should always change if the level data changes                                                                                                          |
| Minecraft Data Version | UInt                                   | [Data version][dataversion-wiki] of the contained level data       |                                                                                                                                                         |
| Metadata               | [Metadata](#metadata)                  | Visual meta information to describe the content                    |                                                                                                                                                         |
| Bounds                 | [Bounds](#bounds)                      | Overall dimensions/size of this SHARD across all sections          |                                                                                                                                                         |
| Config Position Count  | UInt                                   | Amount of Config Positions that are included after this field      |                                                                                                                                                         |
| Config Positions       | [ConfigPosition](#config-position-1)[] | Embedded Config Positions holding positional configuration data    |                                                                                                                                                         |
| Block Palette          | [Palette](#palette)                    | Ordered Palette of contained Block States (referenced in Sections) |                                                                                                                                                         |
| Biome Palette          | [Palette](#palette)                    | Ordered Palette of contained Biomes (referenced in Sections)       |                                                                                                                                                         |
| Section Count          | UInt                                   | Total Amount of Sections that are included within this SHARD       | This is not the amount of Sections that is appended at `Sections`. It is the amount of relevant bits in the mask.                                       |
| Section Mask           | [BitSet](#bitset)                      | Mask to indicate which sections are skipped                        | The BitSet has a length of `Section Count` bits                                                                                                         |
| Sections               | [Section](#section)[]                  | Non-Empty Sections that hold level data for the SHARD              | The amount of sections that are send here is equal to the amount of set bits in `Section Mask`.                                                         |

### UUID

A v4 [UUID](#uuid) that holds total entropy of 128 bits. The raw bits are transferred, as the more readable "normal"
form takes up significantly more space.

| Name                    | Type | Description                                                      | Notes |
|-------------------------|------|------------------------------------------------------------------|-------|
| Most significant bytes  | Long | Most significant (higher) bytes – left bytes of the normal form  |       |
| Least significant bytes | Long | Least significant (lower) bytes – right bytes of the normal form |       |

### Metadata

A metadata object containing the visual metadata that describes the content and origin of the SHARD in a human-readable
form. This can be used to display the SHARD in menus and print summaries.

| Name                      | Type                | Description                                          | Notes                                                                                                                                 |
|---------------------------|---------------------|------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Name Existence            | Boolean             | Whether the `Name` attribute is set                  |                                                                                                                                       |
| Name                      | [String](#string)   | Name of the SHARD                                    | Only present if `Name Existence` is true                                                                                              |
| Description Existence     | Boolean             | Whether the `Description` attribute is set           |                                                                                                                                       |
| Description               | [String](#string)   | Description of the SHARD                             | Only present if `Description Existence` is true                                                                                       |
| Creator Existence         | Boolean             | Whether the `Creator` attribute is set               |                                                                                                                                       |
| Creator                   | [UUID](#uuid-1)     | Creator of the SHARD                                 | Only present if `Creator Existence` is true, A [UUID](#uuid) with only zero bits indicates, that this SHARD was created by the system |
| Creation Moment Existence | Boolean             | Whether the `Creation Moment` attribute is set       |                                                                                                                                       |
| Creation Moment           | [Instant](#instant) | Exact moment, when this SHARD was originally created | Only present if `Creation Moment Existence` is true                                                                                   |

### String

An ordinary string that contains textual information. Strings are always encoded in standard UTF-8 and are prefixed by
the number of bytes that the encoded message needs.

| Name   | Type   | Description                            | Notes |
|--------|--------|----------------------------------------|-------|
| Length | Int    | Amount of bytes for the following text |       |
| Text   | Byte[] | UTF-8 encoded bytes of the text        |       |

### Instant

A single, specific point in time. The time is in UTC (Coordinated universal time) and is based on the UNIX epoch. The
nanoseconds represent the time within the second selected by `Seconds`.

| Name    | Type | Description                     | Notes                  |
|---------|------|---------------------------------|------------------------|
| Seconds | Long | Seconds since unix epoch (1970) |                        |
| Nanos   | Long | Nanos within the current second | Range 0 to 999,999,999 |

### Bounds

An object that represents the bounds or dimensions of something. Minecraft determines the orientation of the x-, y- and
z-axis. Therefore, the y-axis is the height, as usual in Minecraft.

| Name | Type | Description             | Notes |
|------|------|-------------------------|-------|
| X    | Int  | Length along the x-axis |       |
| Y    | Int  | Length along the y-axis |       |
| Z    | Int  | Length along the z-axis |       |

### Config Position

A [ConfigPosition](#config-position) object that represents a single configuration that is attached to a specific
position, relative to the origin of the SHARD.

| Name     | Type                  | Description                                                               | Notes                                             |
|----------|-----------------------|---------------------------------------------------------------------------|---------------------------------------------------|
| Position | [Position](#position) | Relative position within the SHARD                                        | Must be within the bounds of the SHARD            |
| Type     | [String](#string)     | Exact type of the ConfigPosition (what this ConfigPosition represents)    | Multiple ConfigPositions may have the same `Type` |
| Data     | [Mapping](#mapping)   | Mapping of arbitrary key-value configuration data for this ConfigPosition |                                                   |

### Mapping

A mapping object that contains an arbitrary amount of [Entries](#entry) (key-value pairs). A mapping may also be empty
or contain up to 256 entries of varying length. Each entry must be unique.

| Name    | Type              | Description                                 | Notes |
|---------|-------------------|---------------------------------------------|-------|
| Count   | UByte             | Amount of mappings that are present         |       |
| Entries | [Entry](#entry)[] | Individual entries of the key-value mapping |       |

### Entry

An entry object that maps a specific value to a specific key. The key or value may be empty, but a single specific
[Mapping](#mapping) must not contain the same key twice.

| Name  | Type              | Description                   | Notes |
|-------|-------------------|-------------------------------|-------|
| Key   | [String](#string) | Key of the data entry         |       |
| Value | [String](#string) | Value associated with the key |       |

### Position

A position object that contains a relative offset to the origin of the SHARD. All three axes cannot be negative as that
would mean they are not included in the SHARD.

| Name | Type  | Description                                                  | Notes                          |
|------|-------|--------------------------------------------------------------|--------------------------------|
| X    | Float | Offset along the x-axis, relative to the origin of the SHARD | Must be positive (including 0) |
| Y    | Float | Offset along the y-axis, relative to the origin of the SHARD | Must be positive (including 0) |
| Z    | Float | Offset along the z-axis, relative to the origin of the SHARD | Must be positive (including 0) |

### Palette

A palette object that contains the [SNBT](#snbt) entries of the individual entries (blocks or biomes). Each palette has
an independent `Scale` that specifies what datatype the references to this palette (in [Sections](#section)) will use.
For example, if the SHARD has a tiny palette of entries for this kind (up to 256 distinct entries), each reference
can be expressed as a single `UByte`. If the number of distinct entries grows beyond this threshold, `UShort` and
even `UInt` have to be used. Even if the specific index could be expressed in a smaller unit, the palette `Scale`
dictates that all references have to be of the same datatype.

| Name    | Type                | Description                                        | Notes                                                                                                                                                                      |
|---------|---------------------|----------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Size    | UInt                | Count of the palette entries                       |                                                                                                                                                                            |
| Scale   | UByte               | Scale of palette used to encode the entries        | SMALL: `0x00` (UByte – up to 256 distinct entries)<br/>MEDIUM: `0x01` (UShort – up to 65536 distinct entries)<br/>LARGE: `0x02` (UInt – up to 4294967296 distinct entries) |
| Entries | [String](#string)[] | Individual entries of the palette in [SNBT](#snbt) | The number of entries is determined by the Size attribute.                                                                                                                 |

### BitSet

A BitSet is represented as `Byte[]` that has the length of the bits, rounded up to the next multiple of 8. So if there
are, for example, 26 bits, the length in bytes is 32. The padded bits to the right are always `0`. The bit with the
index `0` is the most significant bit of the BitSet. The `length - 1` bit is the least significant bit or near to it,
accounting for padding.

#### Section Mask

In the context of the `Section Mask`, each set bit indicates a section that is present within the data, while an unset
bit indicates a Section that should be skipped and substituted by an empty section. So if there is, for example, a Count
of eight sections, and the BitSet looks like this `00011000`, only two sections will be present (fourth and fifth), and
all other sections should be assumed to be empty.

Only `Complete` sections can be substituted by empty sections. Even if a section is completely empty, it cannot be
omitted (and will therefore always be present and represented by a `1` bit) if it is not `Complete`.

### Section

A section object that stores all data of an individual section within the SHARD. A section has a maximum length of 16
blocks along each axis. That means a section contains at most `16 * 16 * 16 = 4096` blocks. If a section has the maximum
bounds, it is considered `Complete`.

The contained block and biome references are ordered from x, to z, to y. That means for a `Complete` section, that the
block that is at the relative position `(5, 0, 0)` will have index 5, while `(0, 5, 0)` will have the index `1280`.
The data type of the block and biome references is set by the corresponding [palette](#palette).

Sections are required to be `Complete`, if possible with the remaining blocks. That means only the sections that are
at the upper edge of the bounds on any (or multiple) of the three axes could possibly be incomplete. A SHARD with the
size `17 * 17 * 17` would contain one `Complete` section (that's right at the origin) and seven incomplete sections:
{`1 * 16 * 16`, `16 * 16 * 1`, `1 * 16 * 1`, `16 * 1 * 16`, `1 * 1 * 16`, `16 * 1 * 1`, `1 * 1 * 1`}. Sections are
always created starting from the origin.

| Name                 | Type                              | Description                                                           | Notes                                                                                                                                                                                              |
|----------------------|-----------------------------------|-----------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Complete             | Boolean                           | Whether this section is complete                                      | That means the bounds are implicitly 16x16x16                                                                                                                                                      |
| Bounds               | [Bounds](#bounds)                 | Bounds/Dimensions of this section                                     | Only present if `Complete` is false, maximum of 16 for each axis                                                                                                                                   |
| Blocks               | UByte[] \| UShort[] \| UInt[]     | References of the blocks of this section                              | Length equals the three axes of the bounds multiplied. Whether a `UByte`, `UShort` or `UInt` is used for each reference, is set by the `Scale` of the block [palette](#palette). Order is x->z->y. |
| Biomes               | UByte[] \| UShort[] \| UInt[]     | References of the biomes of this section                              | Length equals the three axes of the bounds multiplied. Whether a `UByte`, `UShort` or `UInt` is used for each reference, is set by the `Scale` of the biome [palette](#palette). Order is x->z->y. |
| Block Entities Count | UInt                              | Amount of [block entities](#block-entity-1) included after this field |                                                                                                                                                                                                    |
| Block Entities       | [Block Entity](#block-entity-1)[] | [Block Entities](#block-entity-1) of this section                     |                                                                                                                                                                                                    |
| Entities Count       | UInt                              | Amount of [entities](#entity-1) included after this field             |                                                                                                                                                                                                    |
| Entities             | [Entity](#entity-1)[]             | [Entities](#entity-1) of this section                                 |                                                                                                                                                                                                    |

### Block Entity

A Block Entity object that contains information on a single instance within a [Section](#section). The position is
relative to the origin of the section and must be within the bounds of the specific section.

The position tag `Pos` has to be removed from the [SNBT](#snbt) data, as the position is changed within this SHARD. The
data is instead manually constructed using the new absolute origin with the relative offset.

| Name     | Type                  | Description                                     | Notes                                                         |
|----------|-----------------------|-------------------------------------------------|---------------------------------------------------------------|
| Position | [Position](#position) | Relative position within the section            | Relative to the section origin, must be within section bounds |
| Data     | [String](#string)     | Properties of the block entity in [SNBT](#snbt) |                                                               |

### Entity

An Entity object that contains information on a single instance within a [Section](#section). The position is relative
to the origin of the section and must be within the bounds of the specific section.

The following fields have to be scrapped from the [SNBT](#snbt) as their content generally cannot be transferred to
other worlds or other locations or is not desirable/meaningless in the context of the SHARD format. Instead, they are
taken from the new entity that gets spawned when applying the SHARD to a world:

* `UUID`
* `Pos`
* `Paper.Origin`
* `Paper.OriginWorld`
* `Paper.SpawnReason`
* `Spigot.ticksLived`
* `WorldUUIDMost`
* `WorldUUIDLeast`
* `TileX`
* `TileY`
* `TileZ`

| Name     | Type                  | Description                                                                | Notes                                                         |
|----------|-----------------------|----------------------------------------------------------------------------|---------------------------------------------------------------|
| Position | [Position](#position) | Relative position within the section                                       | Relative to the section origin, must be within section bounds |
| Type     | [String](#string)     | [Resource Location](#resource-location) of the [Entity Type](#entity-type) |                                                               |
| Data     | [String](#string)     | Properties of the entity in [SNBT](#snbt)                                  |                                                               |

## Compression

The bytes may be compressed by [zstd][zstd-github] (or any other compression algorithm) indicated by a matching file
ending or equivalent information. For example, SHARDs that have been compressed by zstd will have a file suffix of
`.shard.zst`. If a file was compressed with gzip, it would be `.shard.gz`. The compression is not part of the data
format and can therefore be swapped and tweaked at the end-implementation's will. Official support of the reference
implementation is limited to [zstd][zstd-github] and `.shard.zst`.

The compression level within each format (if applicable) can be chosen freely. Assumptions must not be made by the
decompiler, and each decompiler is expected to at least support any configuration of zstd compression. Other formats may
be supported.


[shard-format]: README.md

[rfc-2119]: https://www.ietf.org/rfc/rfc2119.txt

[uuid-wiki]: https://wikipedia.org/wiki/Universally_Unique_Identifier

[nbt-wiki]: https://minecraft.wiki/w/NBT_format

[snbt-wiki]: https://minecraft.wiki/w/NBT_format#SNBT_format

[resource-location-wiki]: https://minecraft.wiki/w/Resource_location

[biome-wiki]: https://minecraft.wiki/w/Biome

[block-state-wiki]: https://minecraft.wiki/w/Block_states

[block-entity-wiki]: https://minecraft.wiki/w/Block_entity

[entity-type-wiki]: https://minecraft.wiki/w/Java_Edition_data_values#Entities

[entity-wiki]: https://minecraft.wiki/w/Entity

[entity-format-wiki]: https://minecraft.wiki/w/Entity_format

[byteorder-wiki]: https://en.wikipedia.org/wiki/Endianness

[dataversion-wiki]: https://minecraft.wiki/w/Data_version

[zstd-github]: https://github.com/facebook/zstd
