# SHARD Format Specification

The SHARD (Shard Highly Augmented Region Data) format holds information regarding Minecraft levels that can be either
embedded into existing worlds or loaded as an independent world. It holds auxiliary information that adds meaning
to specific positions within the stored area. Only store basic information that could have an impact on the visual
appearance of the level is stored, omitting technical information that only makes sense for persistent Minecraft worlds.

We developed this format at [JustChunks][justchunks-website] to replace any other Minecraft world representation we
previously used and took inspiration from Hypixel's [Slime format][slime-format-blog] as well as the established
[Sponge Schematic format][sponge-schematic-spec]. The existing options missed some parts that were necessary for our
setup, especially embedded configuration. We wanted SHARDs to be self-sufficient and could be used as standalone sources
for levels including visuals as well as configuration.

Prior to the development of SHARD format v0, we had an inofficial version oriented more closely on the
[Sponge Schematic format][sponge-schematic-spec]. Our levels eventually grew too big to be read in a reasonable
time, and they also consumed way too much memory. That is why we completely scrapped JSON and therefore declared
[human-readability a non-goal](#human-readability) for our next increment as those two factors were limiting to our
pursuit for greater performance.

## Comparison to existing formats

The SHARD format was developed to create a new format that combines the strengths of existing formats and to merge
conventional level data with more sophisticated configuration approaches. Therefore, it may be interesting to compare
our SHARD format to existing, more-established formats available.

### [Minecraft's Region Format (Anvil)][minecraft-anvil-spec]

Minecraft's Region Format is designed to support vanilla Minecraft gameplay and stores a lot of data for those
mechanics. The data is spread across a lot of files, and each region is compressed (by zlib or lzma) individually. This
is poor for the overall size (as a much greater compression rate can be reached) and also involves a lot of unnecessary
operations if the whole world is to be loaded anyway. Additionally, redundant data may form over time and the world
becomes bigger and bigger with no visual differences.

Our SHARD format has little to no extra data and is a single file. It is optionally compressed with zstd and can be
extended with custom configuration data. The SHARD can be loaded either as a world or as a schematic. And it is always
as small as it can be, considering the visual elements that need to be present. We are only using SNBT to store the
individual palettes of block-s, tile entities, and regular entities.

### [Hypixel's Slime Format][hypixel-slime-spec]

Hypixel's Slime Region Format does a lot of things right. It's a binary format, cuts a lot of the unnecessary data,
uses modern compression and can be loaded as a world. It can be serialized and deserialized in a timely manner and only
uses NBT to store the palettes and entities (like we do). There are, however, a few things that we think can be
improved and that we've focused on for the SHARD format. Quite a few of those things may be fixed in future iterations
of the format and could not be predicted when the format was created. Our observations are based on the published
version of the format:

* **(Tile-)Entities are not part of the chunk definitions**: If entities and tile entities were part of the chunk
  definitions, they could be loaded at the moment where the individual chunk is processed. This is great, because that
  means, the chunk won't have to be loaded again (for schematics) and also nice for worlds, as each chunk can be
  constructed in its entirety before going to the next chunk.
* **Chunks are stored from top to bottom with a fixed height**: Modern minecraft worlds can have dynamic heights, which
  is not supported by the Slime Region Format. The format also does not use sections and instead encodes the entire
  chunk in a single block. This wastes potential for optimizations. Most chunks (even in conventional worlds) have a lot
  of empty sections, which the SHARD format can just skip with a single bit. And even the in-memory representation can
  be optimized this way.
* **Biomes are 2D and therefore cannot differ along the y-axis**: Minecraft versions 1.18+ have three-dimensional
  biomes. This means that the biome on the ground can be different from the biome on an isle, floating above it. This
  allows map builders to be more creative with biomes and color shading. The SHARD format has full support for 3D biomes
  and is flexible enough to support any combination of biomes.
* **Can only be loaded as a world, not as a schematic**: Slime files can only be used and loaded as worlds, not as
  schematics. This limits their utility as they cannot be used to assemble complex worlds or to store only sections that
  should be embedded into bigger worlds. The format is optimized only to be used as worlds and does not really fit into
  the concept of schematics or tiny structures.
* **Components are compressed individually**: The individual components of the Slime format are compressed individually.
  This does not only make the serialization/deserialization more complex but also reduces the possible compression
  ratio. The SHARD format compresses the entire file at the end, which has the advantage that compression is optional
  and that we can compress across all data, resulting in optimal results. On top of that, we can stream our data
  asynchronously through multiple layers and perform compression/decompression on-the-fly.
* **Configuration/Extra Data can only be added globally**: Auxiliary data can only be attached globally to the Slime
  file and is written with NBT tags. The SHARD format on the other hand, uses a very minimal binary format and is able
  to store configuration and extra data for any specific position within the SHARD. This allows for better compression
  (smaller memory footprint) and more descriptive configuration that is merged with the visual blocks and regions within
  the contained structure.
* **Does not store the data version**: The Minecraft data version is not saved and cannot be used to efficiently migrate
  the contained data. The version of the NBT data has to be stored outside the Slime file and needs to be kept in sync.
  With SHARDs this data is stored reliably at the beginning so that all following data can be interpreted and migrated
  according to the source and target version.
* **Stores blocks and biomes by their numeric ID**: All blocks and biomes are stored with their dynamic IDs that were
  used before [the flattening][flattening-wiki] happened. Not only are those numeric IDs not used anymore, we also now
  have the ability to add our own biomes and other dynamic data through registries. This is not possible with the Slime
  format. The SHARD format has fully-fledged registry support and only uses temporary IDs with a palette of registry
  keys to take the best of both worlds.

### [Sponge's Schematic Format][sponge-schematic-spec]

Sponge's Schematic format is currently in its third iteration. It supports 3D biomes, can hold any entities, biomes and
blocks but cannot hold any custom configuration. It is based on JSON, making it cumbersome for massive schematics (as
they can be hard to load because of the sheer size) and cannot be used to load them as worlds. Our format was originally
based on the Sponge Schematic Format, but we had to switch away from it, as the runtime performance just became
unbearable for huge worlds.

### [Original Schematic Format (WorldEdit, MCEdit, Schematica)][mcedit-schematic-spec]

The original Schematic Format is based on numeric block IDs and a fixed block palette. It also uses numeric sub IDs and
does not support three-dimensional biomes. It is therefore not possible to represent the data of more recent Minecraft
versions accurately. Those schematics cannot be loaded as worlds, and the format is completely NBT based, which is not
ideal for compression. In general, the original schematic format should not be used for any modern projects.

## Design Principles

During the development of the SHARD format, we agreed on some key design principles that were important for determining
the direction in which we develop the format. Since we have specific requirements for the capabilities and behavior of
our worlds, levels and objects, we abstracted them and extracted some high-level principles from them. These principles
have emerged in particular from the shortcomings of the existing formats and are therefore attempts to address those
inherent problems at the root.

### Atomicity

SHARDs consist of a single, self-contained file with a common state. This simplifies portability, versioning and storage
within cloud-based infrastructures. Having only a single file and therefore only a single source of truth leads to
atomicity. Either the entire world is loaded or nothing is loaded. This prevents faulty or corrupted states, and we can
be sure that if the SHARD is loaded, it is exactly in the state that we expect it to be.

### Immutability

SHARDs are meant to be only read from and write operations always overwrite the entire SHARD. Therefore, SHARDs are
immutable by design. This is also reflected in the data format and is especially beneficial for containerized workloads.
For lobby worlds that cannot be permanently altered by players, writing only wastes resources and is lost on a restart
anyway. By having the SHARDs immutable, we can be sure that a SHARD will always stay the same, no matter what happens on
the server or whether the world has been modified temporarily.

### Extendability

Having a well-defined, rigid format often comes at the cost of extendability. We've made sure that the SHARD format
leaves enough room for domain-specific configuration, metadata, and content. For that, we invented another construct:
*ConfigPosition*. This is a touple of a relative location within the SHARD, a dynamic type and a KV store for auxiliary
data, corresponding to this configuration entry. The format supports up to `2^32 = 4294967296` distinct
*ConfigPositions*, which is more than enough space for any configuration needs.

Those *ConfigPositions* are meant to fuse map configuration with the blocks/entities/biomes themselves. Configuration is
often handled in separate config files. With *ConfigPositions* we merged the configuration into the level data. In the
build worlds those are represented by text displays that are then converted by the integration to *ConfigPositions*.
When the SHARD is loaded as a world, the specific plugin parses those positions and links them with their respective
gameplay significance.

The palettes are also independent of the vanilla registries and can therefore support modded/custom content and custom
data fields for the individual blocks as well. The meaning of the SHARDs is interpreted by the individual integrations.
As long as an integration can understand the specific palette entries, it is completely fine to add custom entries that
may not be part of the official vanilla registries.

### Simplicity

The Minecraft Region Format (Anvil) contains loads of data that we don't need for lobbies, maps, and other use cases for
the SHARD format. This includes stuff like advancements, structure information, points of interest, statistics, time
inhabited in specific chunks, last time a chunk was updated, weather ... it just does not end. All this data is
justified in the context of vanilla Minecraft gameplay, but becomes completely redundant for modern Minecraft servers
with more sophisticated concepts that don't use the default Minecraft mechanics.

The SHARD format is reduced to the bare minimum to represent the map visually. Everything not related to the appearance
of the map is optional, and everything is focused on custom gameplay mechanics. This makes the SHARD format
straightforward to understand and gives all available knobs significance within the context of representing a map or
structure within Minecraft.

To simplify the recognition of SHARD files, we've added our own signature at the start of the files, so that they can
be properly differentiated from other files. That's also why we use a custom file ending, namely `.shard` and for
compressed SHARDs `.shard.zst` to make it very transparent what the content of a SHARD is.

### Flexibility

Perhaps the aspect that stands out the most is that SHARDs can be loaded as worlds **and** schematics, without changing
the file itself. As outlined above: SHARDs just hold level data and configuration. Whether that information is loaded
without anything else (world) or on top of existing structures (schematic) is up to the user. While loaded as a world,
the SHARD format supplies all the necessary visual information and creates the necessary world configuration from the
sensitive defaults, but it is also customizable (pvp, mob spawning, world name, …).

The configuration is completely usage agnostic and therefore not opinionated. Whether the SHARD format is used to
represent a lobby, a Bedwars map, a clan base or the template for a plot, they all fit into the SHARD format. This can
also be seen by the sheer size that the format theoretically can support. It holds up to `2^32` sections of `16^3`
blocks. That means that absolutely all custom worlds should be able to be stored in the SHARD format.

And it is also forward-compatible: By using Mojang's [DataFixerUpper (DFU)][dfu-website], SHARDs can be upgraded to
any more recent Minecraft version, if the migration is present in the runtime environment. The version used to store the
SHARD is persisted along the content, so it's straightforward to track what migrations have to be performed on the
palettes and entity information.

### Efficiency

Speed is important in modern gaming. Players don't want to wait for their maps to be assembled, minigames require
frequent loading and deserialization of map data and maps have to be regularly updated for new features. Therefore,
efficiency and performance were a major concern during the development of the SHARD format. The inofficial pre-version
of the SHARD format suffered from choosing JSON to structure the data, especially with huge worlds (~3000x3000 blocks).
Therefore, we chose to implement our own binary format for SHARDs v0.

The result is blazingly fast and very space efficient. The smallest possible SHARD is only 73 bytes (33 bytes if
compressed), and everything from the bottom up was built with performance in mind. Serialization and deserialization are
done with async I/O and native channels, the data is distributed in sections to enable concurrency and allow for "empty"
data to be culled. The sections are prefixed by a bitmask that reduces the space consumption and parsing of empty
sections to just a single bit. Also, the in-memory representation uses a singleton to represent empty sections to
improve the runtime memory usage and equality check performance.

Compression is entirely optional, and it can be beneficial for the read performance of very small SHARDs to disable it,
but the default is [zstd][zstd-github]. Other compression algorithms can be used und the compression is not part of the
file format itself. This is reflected by the file ending being `.shard` for non-compressed SHARDS and `.shard.zst` for
SHARDs that have been compressed with zstd.

Although we support updating Minecraft data ad-hoc, we skip migrations altogether if the detected Minecraft data version
is equal to the data version detected within the runtime. This saves a lot of time, so we migrate our SHARDs whenever a
new Minecraft version is released to not rely on the ad-hoc migration. This means we'll only have to execute the
migration once and not again and again, every time the map is loaded.

## Non-Goals

We consider certain aspects as out of the scope for the SHARD format. Those are things that don't fit well with the rest
of the format, are a hindrance to the development of other functionality, make things unnecessarily complex or just come
with too much of a maintenance burden. That's why we're confident they will never be a part of the SHARD format.

### Replace vanilla Minecraft worlds

The SHARD format is not meant to replace normal Minecraft (vanilla) worlds. We don't try to mimic or adapt normal
Minecraft mechanics, introduce permanent mutability or auxiliary data like statistics, advancements and structures.
Therefore, the SHARD format may not be a good fit for all servers that want to offer more vanilla oriented gameplay.
This includes servers that rely on the default implementations of advancements, statistics or other data that can be
normally found in Minecraft worlds.

### Support older versions

It's also not a goal for the SHARD format to support Minecraft versions prior to ["the Flattening"][flattening-wiki]
(Version 1.13). The palettes of more recent versions are optimized for the (S)NBT representations of blocks and the
previous versions referred to block types by a numeric ID and another numeric sub ID. Supporting those legacy formats
would incur a lot of complexities on our data schema as well as the code for transformations. Therefore, we will always
only support the latest version of Minecraft's internal data scheme, but provide upgrade paths.

### Human Readability

As the SHARD format is a binary format, human readability cannot be achieved. Instead, we rely on tooling to allow
users to inspect the contents of SHARDs and their metadata. SHARDs are not meant to be edited by hand (as their schema)
could be easily messed up, if manually modified. Therefore, the loss of human readability in the raw format is not a
great loss. The same applies for debugging, which is why we don't add any debug data to the SHARDs.

## Transformation

One of the key advantages of the SHARD format is the ability to perform swift and efficient transformations on the data.
Transformations within real Minecraft worlds have to account for various locks (concurrency), multiple distributed data
locations, states, packets and other components because the world is already loaded. SHARDs, on the other hand, can be
transformed purely mathematically and data-driven. This is time and space efficient and allows us to first get the
SHARDs in the desired shape before we apply them to the Minecraft worlds.

### Merge (Layering)

A very convenient operation we can perform on SHARDs is the layering or merge. This means overlaying multiple SHARDs in
a specific order and with different offsets to create a new combined SHARD that can be integrated into the Minecraft
server in a single step. The resulting SHARD is optimized to only be as big as strictly necessary, and (now) empty
sections are culled from the final result. The transformation supports an arbitrary number of SHARDs to be merged on top
of another with positive and negative offsets. The SHARD is grown to support all blocks at their respective offsets, and
entities can be purged on lower layers or just added to the existing entities.

### Rotation

Sometimes the orientation of SHARDs does not match the orientation that they should have in the world. Shards can be
rotated around all three axes in steps of 90 degrees. While other angles would also be possible, this is outside the
scope of SHARDs, as that would require some kind of interpolation, and we focus on clean transformations. The blocks in
the palette are also rotated accordingly (if possible for this angle) and the dimensions of the SHARD are adjusted to
match the new orientation.

### No-Op

The No-Op transformation just returns the original SHARD. This is useful if the real transformations should be
temporarily disabled or skipped without adjusting the whole code. This way, the real transformation is just replaced
with the No-Op transformation and everything still works as expected.

## Integration

Without integration into the Minecraft ecosystem, SHARDs would not serve any purpose. Through the use of Mojang's
[DataFixerUpper (DFU)][dfu-website], the palettes and entities within SHARDs can already be migrated ad-hoc to any
desired Minecraft version. After they have been migrated to the requested version, they may be loaded into the Minecraft
server: Either as an independent world or as a part of an existing world (schematic).

### As a World

Loading SHARDs as world is done through a patch in our [Paper][paper-website] fork Cardboard. SHARD worlds are loaded
as natively as possible and are injected into the heart of the Minecraft server. The world loading is highly optimized,
and all chunks are kept loaded all the time. The chunks outside the defined area are replaced with a special empty
placeholder chunk that is neither ticked nor sent to the players. This improves the server performance and is also
beneficial for the traffic and client performance of each player.

We've also made sure that SHARDs can be loaded as the main world and are compatible with any third-party plugins (except
for those that access the world folder on their own). Loading worlds in this native way allows us to stay compatible
with the existing API and only offer extensions on top of it without having to establish a new ecosystem. SHARDs can
also be used alongside normal vanilla worlds and are therefore completely compatible with most workloads.

### As a Schematic

Embedding and extracting a SHARD as a schematic (that means as a piece of an existing world) happens through optimized
integrations. Those integrations either scan the world for its contents and store the discovered information in our
SHARD format, or they take the existing information from the SHARD and apply (overwrite) the content in the world. While
iterating over the contents, it is possible to specify transformations and filters that influence how or if specific
contents are applied to the world or SHARD.

The SHARDs can be inserted at any coordinate, and all data within the SHARD is transposed to that coordinate. This also
applies to config positions and entities. It can be modified whether existing blocks and entities will be purged before
inserting the SHARD or if only air will be replaced and the entities will be added alongside the existing inhabitants of
the world. This can be controlled by the use of filters.

## Versioning

To add new functionality and allow for optimizations or adjustments, the concept of different versions or iterations was
considered from the very start of the specification. Versions are meant to be improvements to previous increments, so
there should not be any reason to prefer an old version over a newer version.

### Scheme

A new version will be published each time a breaking change (that requires adjustments to either the serialization,
deserialization or integration) is necessary. They are neither forward- nor backwards-compatible, but the SHARD format
library never drops old versions and will try to deserialize each SHARD in its specified version. The version is always
incremented to the next integer.

Optimizations (that do not change the underlying data) do not trigger version bumps. So if old unoptimized
implementations still are able to serialize and deserialize SHARDs, optimizations can happen at any time. This could,
for example, be an optimization to the system calls involved in loading the file or some internal caching that speeds up
the read process.

### Changelog

To allow an easier distinction between the different changes from one version to another and the necessary migration,
this table holds all published versions of the SHARD format so far. Once a new version is released, a new specification
is added to this repository, and it is referenced within the table below.

| Version          | Name    | Date       | Note            |
|------------------|---------|------------|-----------------|
| [0](shard-v0.md) | initial | 2023-06-17 | Initial version |

## Optimization Possibilities

Even though we've gone to great lengths to optimize the SHARD format as much as possible, there are still a few problems
we were not yet able to solve. Some of them are tricky, and there is no obvious solution, others have solutions that
come with their own drawbacks. We will try to solve those problems in future iterations of the SHARD format and strive
to optimize it even further! Below is a list of optimization possibilities that we currently track:

### 3D Biomes save unnecessary data

Minecraft 1.18 introduced 3D biomes. This means that biomes are no longer guaranteed to be the same across all y levels,
but instead can have different biomes at different heights. The SHARD format supports this information, but there is a
small overhead in how we save that data. We save biome information for **every** block, although only every 16th block
would be necessary. This is because biomes are saved for 4x4x4 cubes.

We chose to probe and store the biome for each block anyway, as the offsets when pasting SHARDs as a schematic or
loading it as a world can cause the biome grid to shift. Therefore, the more accurate data can help to interpolate which
biome should be where. If we find a better suited algorithm, we could cut down on the biome data within the stored
SHARDs and therefore reduce our memory footprint.

### Bounds for Sections could be saved as bytes

Bounds for individual sections are saved as `int`, if the sections are not *complete* (16x16x16). This is often the
case for the edge sections towards the upper bounds of the SHARD, as the total dimension of the SHARD is not a multiple
of 16 for all axes. In these cases, we are wasting nine bytes per incomplete section, as the maximum size of a section
is 16, which can comfortably fit into a byte. In fact, we could even store the bounds in 12 bits (3 nibbles) if we
really wanted to squash as much data as possible.

The reason why we still store the bounds as ints is because the conversion from byte to int would incur some cost for
each section that we deserialize/serialize. Unless we find a more performant way to convert the types (or have more
thorough benchmarks of the impact), we continue to store them as int. That way, we can use the same type as the
in-memory representation of the bound axes.

### NBT Data could be expressed in binary

We currently save all NBT data as SNBT (so the stringified form). That takes more space than the binary version of the
same information in NBT would take. By serializing to binary instead, we could save both space and processing time.
Considering there's possibly a lot of NBT data (entities, block entities), this could be a considerable optimization
for the SHARD format.

We made this decision to write easier migrations across different data versions and to be able to debug problems with
the stored data more easily. But our format is already binary, so it would be possible (and probably desirable) to
represent NBT Compounds in binary as well. It is, however, crucial that the interfaces remain intuitive and that
efficient encoders/decoders are used for the serialization and deserialization of NBT.

### Light Data could be cached

To speed up world loading within the Minecraft server, we could pre-populate light data (sky- and blocklight) and use
this data to load the world. This only takes a little amount of additional space (which we could even make optional) but
could possibly drastically improve the world loading performance. In theory, this could even be merged into the world
when embedding SHARDs as schematics, rendering most (if not all) light updates redundant.

Including light data in SHARDs increases the complexity of most transformation operations and especially the merging of
SHARDs. To reliably modify light data during those operations, we would need to implement details about how light data
is populated. That is a huge maintenance burden. Alternatively, we could just drop any light data once a modification
has been applied to the SHARD.


[justchunks-website]: https://justchunks.net/

[paper-website]: https://papermc.io/

[slime-format-blog]: https://hypixel.net/threads/dev-blog-5-storing-your-skyblock-island.2190753/

[hypixel-slime-spec]: https://hypixel.net/threads/dev-blog-5-storing-your-skyblock-island.2190753/

[sponge-schematic-spec]: https://github.com/SpongePowered/Schematic-Specification

[minecraft-anvil-spec]: https://minecraft.wiki/w/Anvil_file_format

[mcedit-schematic-spec]: https://minecraft.wiki/w/Schematic_file_format

[zstd-github]: https://github.com/facebook/zstd

[flattening-wiki]: https://minecraft.wiki/w/Java_Edition_1.13/Flattening

[dfu-website]: https://github.com/Mojang/DataFixerUpper
