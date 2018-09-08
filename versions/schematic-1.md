# Sponge Schematic Specification

#### Version 1

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://www.ietf.org/rfc/rfc2119.txt).

## Introduction

This specification defines a format which describes a region of a [Minecraft](http://minecraft.net) world for the purpose of serialization and storage of the region to disk. It is designed in order to allow maximum cross-compatibility between platforms, versions, and various states of modification.

## Revision History

Version | Date | Notes
--- | --- | ---
1 | 2016-08-23 | Initial Version


## Definitions

##### <a name="defBlockState"></a>Block State
A Block State is an instance of a block type with a set of extra data used to further define the material. The additional data varies by block type and a complete listing of vanilla types is available [here](http://minecraft.gamepedia.com/Block_states).

##### <a name="defResourceLocation"></a>Resource Location
Resources identified by a Resource Location have an identifier string which is made up of two sections, the domain and the resource name, written as `domain:name`. If the domain is not specified then it is assumed to be `minecraft`. For example `planks` would identify the same resource as `minecraft:planks` as the `minecraft:` prefix is implicit.

## Specification

### Format

The structure described by this specification is persisted to disk using the [Named Binary Tag](http://minecraft.gamepedia.com/NBT_format) (NBT) format. Before writing to disk the NBT data must be compressed using the [GZip](https://www.gnu.org/software/gzip/) data compression algorithm. The highly recommended file extension for files using this specification is `.schem` (chosen so as to not conflict with the legacy `.schematic` format allowing easy distinction between the two).

All field names in the specification are **case sensitive**.

### Schema

#### <a name="schematicObject"></a>Schematic Object

This is the root object for the specification.

##### Fields

Field Name | Type | Description
---|:---:|---
<a name="schematicVersion"></a>Version | `integer` | **Required.** Specified the format version being used. It may be used to provide validation and auto-conversion from older versions. The current version is `1`.
<a name="schematicMetadata"></a>Metadata | [Metadata Object](#metadataObject) | Provides optional metadata about the schematic.
<a name="schematicWidth"></a>Width | `unsigned short` | **Required.** Specifies the width (the size of the area in the X-axis) of the schematic.
<a name="schematicHeight"></a>Height | `unsigned short` | **Required.** Specifies the height (the size of the area in the Y-axis) of the schematic.
<a name="schematicLength"></a>Length | `unsigned short` | **Required.** Specifies the length (the size of the area in the Z-axis) of the schematic.
<a name="schematicOffset"></a>Offset | `integer`[3] | Specifies the relative offset of the schematic. This value is relative to the most minimal point in the schematics area. The default value if not provided is `[0, 0, 0]`. This may be used when incorporating the area of blocks defined by this schematic to a larger area to place the blocks relative to a selected point.
<a name="schematicPaletteMax"></a>PaletteMax | `integer` | Specifies the size of the block palette in number of bytes needed for the maximum palette index. Implementations may use this as a hint for the case that the palette data fits within a datatype smaller than a 32-bit integer that they may allocate a smaller sized array.
<a name="schematicPalette"></a>Palette | [Palette Object](#paletteObject) | Specifies the block palette. This is a mapping of block states to indices which are local to this schematic. These indices are used to reference the block states from within the [BlockData array](#schematicBlockData). It is recommeneded for maximum data compression that your indices start at zero and skip no values. The maximum index cannot be greater than [`PaletteMax - 1`](#schematicPaletteMax). While not required it is **highly** recommended that you include a palette in order to tightly pack the block ids included in the data array.
<a name="schematicBlockData"></a>BlockData | `varint[]` | **Required.** Specifies the main storage array which contains `Width * Height * Length` entries. Each entry is specified as a varint and refers to an index within the [Palette](#schematicPalette). The entries are indexed by `x + z * Width + y * Width * Length`.
<a name="schematicTileEntities"></a>TileEntities | [TileEntity Object](#tileEntityObject)[] | Specifies additional data for blocks which require extra data. If no additional data is provided for a block which normally requires extra data then it is assumed that the TileEntity for the block is initialized to its default state.

#### <a name="metadataObject"></a> Metadata Object

An object which provides optional additional meta information about the schematic. The fields outlined here are guidelines to assist with standardization but it is recommended that any program reading and writing schematics persist all fields found within this object.

##### Fields

Field Name | Type | Description
---|:---:|---
<a name="metadataName"></a>Name | `string` | The name of the schematic.
<a name="metadataAuthor"></a>Author | `string` | The name of the author of the schematic.
<a name="metadataDate"></a>Date | `long` | The date that this schematic was created on. This is specified as milliseconds since the Unix epoch.
<a name="metadataRequiredMods"></a>RequiredMods | `string`[] | An array of mod ids which have blocks which are referenced by this schematic's defined [Palette](#schematicPalette).

##### Metadata Object Example:

```js
{
    "Name": "My Schematic",
    "Author": "Author Name",
    "RequiredMods": [
        "a_mod",
        "another_mod"
    ]
}
```

#### <a name="paletteObject"></a>Palette Object

An object which holds a mapping of a block state id to an index. The indices are recommended to start at `0` and may be no more than [`PaletteMax - 1`](#schematicPaletteMax). If the palette is not specified then the global mapping of ids is used instead. This is not recommended as it both expands the size of your blockdata and removes cross server and cross version compatibility for modded blocks which may change ids due to a variety of reasons.

#### Block State Ids

The format of the Block State identifier is the id of the block type and a set of comma-separated property `key=value` pairs surrounded by square brackets. If the block has no properties then they can be excluded. The block type id is specified as a [Resource Location](#defResourceLocation).

For example the air block has no properties so its id representation would be just the block type id `minecraft:air`. The planks block however has an enum property for the `variant` so its id would be `minecraft:planks[variant=oak]`. Properties should be ordered with their keys in alphabetical ordering.

##### Fields

Field Pattern | Type | Description
---|:---:|---
<a name="paletteEntry"></a>{blockstate} | `integer` | A single entry mapping a blockstate to an index.

##### Palette Object Example

```js
"Palette" {
    "minecraft:air": 0,
    "minecraft:planks[variant=oak]": 1,
    "a_mod:custom": 2
}
```

#### <a name="tileEntityObject"></a>Tile Entity Object

An object to specify a tile entity which is within the region. Tile entities are used by Minecraft to store additional data for a block (such as the lines of text on a sign). The fields used to describe a tile entity vary for each type, however the structure will be the same as used by the [Minecraft Chunk Format](http://minecraft.gamepedia.com/Chunk_format#Block_entity_format).

##### Fields

Field Pattern | Type | Description
---|:---:|---
<a name="tileEntityVersion"></a>ContentVersion | `integer` | **Required.** A version identifier for the contents of this tile entity. Used for providing better backwards compatibility.
<a name="tileEntityPos"></a>Pos | `integer`[3] | **Required.** The position of the tile entity relative to the `[0, 0, 0]` position of the schematic (without the [offset](#schematicOffset) applied). Must contain exactly 3 integer values.
<a name="tileEntityId"></a>Id | `string` | **Required.** The id of the tile entity type defined by this Tile Entity Object, specified as a [Resource Location](#defResourceLocation). This should be used to identify which fields should be required for the definition of this type.

##### Tile Entity Object Example

An example of possible storage of a sign. See the [Minecraft Chunk Format](http://minecraft.gamepedia.com/Chunk_format#Block_entity_format) for a complete listing of data used to store various types of tile entities present in vanilla minecraft. Mods may store additional data or have additional types of tile entities.

```js
{
    "ContentVersion": 0,
    "Pos": [0, 1, 0],
    "Id": "minecraft:Sign",
    "Text1": "foo",
    "Text2": "",
    "Text3": "bar",
    "Text4": ""
}
```
