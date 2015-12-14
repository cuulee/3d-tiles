# Instanced 3D Model

## Contributors

* Sean Lilley, [@lilleyse](https://twitter.com/lilleyse)
* Patrick Cozzi, [@pjcozzi](https://twitter.com/pjcozzi)

## Overview

_Instanced 3D Model_ is a tile format for efficient streaming and rendering of a large number of models, called _instances_, with slight variations.  In the simplest case, the same tree model, for example, may be located - or _instanced_ - in several places.  Each instance references the same model, and has per-instance properties, such as position.

In addition to trees, Instanced 3D Model is useful for fire hydrants, sewer caps, lamps, traffic lights, etc.

A Composite tile can be used to create tiles with different types of instanced models, e.g., trees and traffic lights.

Instanced 3D Model maps well to the [ANGLE_instanced_arrays](https://www.khronos.org/registry/webgl/extensions/ANGLE_instanced_arrays/) extension for efficient rendering with WebGL.

## Layout

A tile is composed of a header immediately followed by a body.

**Figure 1**: Instanced 3D Model layout (dashes indicate optional sections).

![](figures/layout.png)

## Header

The 28-byte header contains:

|Field name|Data type|Description|
|----------|---------|-----------|
| `magic` | 4-byte ANSI string | `"i3dm"`.  This can be used to identify the arraybuffer as an Instanced 3D Model tile. |
| `version` | `uint32` | The version of the Instanced 3D Model format. It is currently `1`. |
| `byteLength` | `uint32` | The length of the entire tile, including the header, in bytes. |
| `batchTableByteLength` | `uint32` | The length of the batch table in bytes. Zero indicates there is not a batch table. |
| `gltfByteLength` | `uint32` | The length of the glTF field in bytes. |
| `gltfFormat` | `uint32` | Indicates the format of the glTF field of the body.  `0` indicates it is a url, `1` indicates it is embedded binary glTF.  See the glTF section below. |
| `instancesLength` | `uint32` | The number of instances. |

_TODO: code example reading header_

If either `gltfByteLength` or `instancesLength` equal zero, the tile does not need to be rendered.

The body immediately follows the header, and is composed of three fields: `Batch Table`, `glTF`, and `instances`.

## Batch Table

_TODO: create a separate Batch Table spec that b3dm, i3dm, etc. can reference?_

In the Binary glTF section, each vertex has an unsigned short `batchId` attribute in the range `[0, number of models in the batch - 1]`.  The `batchId` indicates the model to which the vertex belongs.  This allows models to be batched together and still be identifiable.

The batch table is a `UTF-8` string containing JSON.  It immediately follows the header.  It can be extracted from the arraybuffer using the `TextDecoder` JavaScript API and transformed to a JavaScript object with `JSON.parse`.

The JavaScript object contains a set of properties with pre-defined names. Each property is an array with its length equal to `header.batchLength` so that values from the batch table can be matched with the models' `batchId` attribute.

A property with name "id" provides unique identifiers, which can used in hash tables or Javascript associative arrays. The array contains `String` or `Number` elements.  Elements may be `null`.

A property with name "attributes" is an array of JSON objects containing key values pairs. The key provides the attribute's name. The value can be of type `String`, `Number`, 'Boolean', or a JSON object.  Elements may be `null`.

None of the properties are mandatory. If no information is available, an empty JSON object is also valid.

The JavaScript object can be extended by custom specific properties, each containing an array with its length equal to `header.batchLength`.


A vertex's `batchId` is used to access elements in each array and extract the corresponding properties.  For example, the following batch table has properties for a batch of three instances.
```json
{
    "id" : ["Tree_39251","Tree_39252","Tree_39253" ],    
    "attributes" : [{"displayName":"Small Tree","species":"Acer platanoides"},{"displayName":"Old Oak"},{"species":"Quercus robur", "height":16.5} ]
}
```

The properties for the instance with `batchId = 0` are
```javascript
id[0] = 'Tree_39251';
attributes[0]['displayName'] = 'Small Tree';
attributes[0]['species'] = 'Acer platanoides';
```

The properties for `batchId = 1` are
```javascript
id[1] = 'Tree_39252';
attributes[1]['displayName'] = 'Old Oak';
```

The properties for `batchId = 2` are
```javascript
id[2] = 'Tree_39253';
attributes[2]['species'] = 'Quercus robur';
attributes[2]['height'] = 16.5;
```


## glTF

The glTF field immediately follows the batch table (or immediately follows the header, if `header.batchTableByteLength` is zero).

[glTF](https://www.khronos.org/gltf) is the runtime asset format for WebGL.  [Binary glTF](https://github.com/KhronosGroup/glTF/tree/master/extensions/Khronos/KHR_binary_glTF) is an extension defining a binary container for glTF.  Instanced 3D Model uses glTF 1.0 with the [KHR_binary_glTF](https://github.com/KhronosGroup/glTF/tree/master/extensions/Khronos/KHR_binary_glTF) extension.

`header.gltfFormat` determines the format of the glTF field.  When it is `0`, the glTF field is

* a UTF-8 string, which contains a url to a glTF model.

When the value of `header.gltfFormat` is `1`, the glTF field is

* a binary blob containing binary glTF.

In either case, `header.gltfByteLength` contains the length of the glTF field in bytes.

## Instances

The `instances` field immediately follows the `glTF` field (which may be omitted when `header.gltfByteLength` is `0`).

The `instances` field contains `header.instancesLength` of tightly packed instances.  Each instance has three fields:

|Field name|Data type|Description|
|----------|---------|-----------|
| `longitude` | `double` | The longitude, in radians, in the range `[-PI, PI]`. |
| `latitude` | `double` | The latitude, in radians, in the range `[-PI / 2, PI / 2]`. |
| `batchId` | `uint16`  | ID in the range `[0, length of arrays in the Batch Table)`, which indicates the corresponding properties. |

_TODO: make this much more memory efficient and more general._

When `header.batchTableByteLength` is zero, which indicates there is not a batch table, `batchId` is omitted, so each instance contains only `longitude` and `latitude` fields.

Each instance is in the east-north-up reference frame (`x` points east, `y` points north, and `z` points along the geodetic surface normal).

## File Extension

`.i3dm`

## MIME Type

_TODO_

`application/octet-stream`
