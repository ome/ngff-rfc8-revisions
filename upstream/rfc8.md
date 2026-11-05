# RFC-8: Collections and Extensibility

```{toctree}
:hidden:
:maxdepth: 1
comments/index
versions/index
```

Extending OME-Zarr with new metadata types, references, and collections

## Status

This proposal is early. Status: D1

| Name      | GitHub Handle | Institution | Date       | Status                                |
| --------- | ------------- | ----------- | ---------- | ------------------------------------- |
| Norman Rzepka | [normanrz](https://github.com/normanrz) | scalable minds | 2024-11-20 | Author |
| Eric Perlman | [perlman](https://github.com/perlman) | Yikes LLC | 2024-11-20 | Author |
| Joel Lüthi | [jluethi](https://github.com/jluethi) | BioVisionCenter Zurich | 2024-11-20 | Author |
| Lorenzo Cerrone | [lorenzocerrone](https://github.com/lorenzocerrone) | BioVisionCenter Zurich | 2024-11-20 | Author |
| Johannes Soltwedel | [jo-mueller](https://github.com/jo-mueller) | German BioImaging e.V. | 2025-10-28 | Author |
| Christian Tischer | [tischi](https://github.com/tischi) | EMBL | 2025-02-01 | Contributor |
| Matthew Hartley | [matthewh-ebi](https://github.com/matthewh-ebi) |  EMBL-EBI | 2025-05-05 | Contributor |
| Damien Goutte-Gattat | [gouttegd](https://github.com/gouttegd) | German BioImaging e.V. | 2026-10-22 | Author |

<!--
| Author    | N/A           | N/A         | xxxx-xx-xx | Author; Implemented (link to release) |
| Commenter | N/A           | N/A         | xxxx-xx-xx | Endorse (link to comment)             |
| Commenter | N/A           | N/A         | xxxx-xx-xx | Not yet (link to comment)             |
| Endorser  | N/A           | N/A         | xxxx-xx-xx | Endorse (no link needed)              |
| Endorser  | N/A           | N/A         | xxxx-xx-xx | Implementing (link to branch/PR)      |
| Reviewer  | N/A           | N/A         | xxxx-xx-xx | Endorse (link to comment)             |
| Reviewer  | N/A           | N/A         | xxxx-xx-xx | Requested by editor                   |-->

## Overview

<!--
The RFC begins with a brief overview. This section should be one or two
paragraphs that just explains what the goal of this RFC is going to be, but
without diving too deeply into the "why", "why now", "how", etc. Ensure anyone
opening the document will form a clear understanding of the RFCs intent from
reading this paragraph(s).

-->

This proposal introduces a general extensibility mechanism for OME-Zarr
metadata. It defines a common Node interface, a system for referencing nodes
locally and remotely, and an extension scheme that allows OME-Zarr to be extended
with new node types and metadata while maintaining interoperability.

The mechanism also provides a general way to represent collections: groups of
images and other data objects that can be nested, referenced, and enriched with
additional metadata. Existing concepts such as multiscales, labels, HCS
layouts, and coordinate transformations can be represented within this
framework, while future extensions can introduce additional data types and use
cases.

This proposal does not aim to define all possible extensions or metadata types.
Instead, it establishes the building blocks and extension points through which
OME-Zarr can evolve in a controlled and interoperable manner.

## Background

Scientific imaging increasingly involves collections of related data rather
than isolated images. A single experiment may produce multiple images of the
same sample, images acquired with different modalities or from different views,
derived images such as segmentations and prediction maps, and additional data
such as tables or meshes. These data may have relationships that are important
for interpreting them: they may share a coordinate space, originate from the
same acquisition, represent different stages of a processing workflow, or
need to be visualized together.

OME-Zarr already provides mechanisms for several of these use cases. For
example, labels can be associated with images, high-content screening data can
be organized into plates and wells, and RFC-5 provides coordinate systems and
transformations for relating images in a common coordinate space. The
bioformats2raw.layout metadata provides another way of organizing series of
images. However, these mechanisms have developed around individual use cases
and consequently provide different structures and conventions for representing
related data.

At the same time, applications have developed their own metadata formats for
grouping images and describing their relationships. Viewers such as Webknossos,
Neuroglancer, MoBIE and OMERO.figure can display multiple images together and
have developed JSON-based metadata to describe those collections, including
path references, coordinate transformations and rendering settings. Workflow
systems may need to associate input images with derived outputs without
modifying the original data. Archives need to describe collections of images
during both deposition and subsequent access. Similar needs arise for gallery
and grid views, correlative imaging, and images published at remote locations.

This fragmentation makes it difficult to exchange collections of scientific
data between tools. Users may need to maintain tool-specific metadata for the
same underlying relationships, while archives and other data providers may need
to produce multiple representations of the same collection for different
consumers. A general mechanism for describing related objects would allow these
use cases to be represented consistently while leaving application-specific
metadata to the applications that need it.

Extensibility is important because the range of scientific data and workflows
cannot be anticipated in a single specification. In addition to images and
segmentations, workflows may produce prediction maps, tables, meshes and other
derived data. Similarly, different applications may need to attach metadata
describing rendering state, processing context, or other relationships between
objects. Rather than defining a separate top-level mechanism for every new use
case, OME-Zarr should provide well-defined extension points through which new
node types and metadata can be introduced while maintaining interoperability.

Several existing standards could potentially be used to represent such
collections. For example, Research Object Crate (RO-Crate) provides a general
mechanism for describing collections of research artifacts and their
relationships, while JSON-LD provides standardized mechanisms for types,
identifiers and extensibility. These approaches provide useful capabilities,
but they also introduce requirements and complexity that are not well matched
to the requirements of OME-Zarr. In particular, path-based references to Zarr
and JSON metadata, including relative paths and different path types, are
central to this proposal. A purpose-built mechanism can retain the
human-readable and storage-oriented characteristics of OME-Zarr while providing
the extensibility needed by its users.

The goal of this proposal is therefore not simply to introduce another
collection format. It is to provide a common and extensible foundation for
representing OME-Zarr objects and their relationships. Collections are an
important application of this foundation: they allow images and other objects
to be grouped, nested, referenced locally or remotely, and enriched with
additional metadata. The same mechanism can also provide a path for
incorporating existing OME-Zarr structures and future data types into a more
consistent framework.

## Proposal

The following sections describe the proposed metadata framework in detail. They
define the building blocks, the abstract structure, and the core classes in
order. Each section includes schemas and small illustrative examples intended
to explain the structure and relationships between the different components.

The examples in these sections are deliberately minimal and focus on
illustrating individual aspects of the proposal, e.g.:


```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "multiscale",
        "id": "01a13520-b697-70c4-979b-b52c01fbbe30",
        "name": "our_first_example",
        "nodes": [
          {
            "id": "01a13da0-6c3b-7600-b6f5-4ef2414b2f14",
            "name": "s0",
            "type": "singlescale",
            "path": {
              "type": "zarr",
              "format": "relative",
              "href": "./s0"
            },
          }
        ]
    }
}
```

More complete examples showing how these components can be combined to support
the use cases described in the Background are provided later in the document.

### Features deferred to subsequent sub-RFC

This RFC will defer the definition of some features to forthcoming sub-RFCs. 
In doing so, we aim to help reviewers focus on providing feedback on the core building being defined in this RFC. 
In parallel to the review of RFC-8, we will author sub-RFCs for each deferred feature that propose how they will be expressed using the core RFC-8 building blocks. 
As we iterate on the design of RFC-8, we will update the sub-RFCs. 
All deferred features will be proposed and evaluated via the RFC process before the v1.0 specification is released.

The following features are being deferred to sub-RFCs:

| Topic | Scope |
|---|---|
| **HCS** | Plate and well traits |
| **Labels** | Label image traits and per-label properties |
| **bioformats2raw and OME-XML** | Series layout and the OME/METADATA.ome.xml mapping |
| **Absolute and remote references** | Absolute paths and URLs, authentication and security |
| **snake_case** | Making keys and identifiers consistent with Zarr |
| **Legacy multiscales** | A `legacy_multiscale` node type for unchanged 0.5 and 0.6 images |
| **Singlescale images** | A node type for images with a single resolution level. |

### Building blocks

The building blocks define how objects are represented using paths, declarations and references.

(rfcs:rfc8:path-interface)=
#### `Path` interface

This new interface replaces the paths defined in the previous versions of the
OME-Zarr specification.

(rfcs:rfc8:path-interface-example)=
##### Example

From our first example, this is the lowest level type
used to build up the definition of an image.

```jsonc
    "path": {
        "type": "zarr",
        "format": "relative",
        "href": "./s0"
    }
```

(rfcs:rfc8:path-interface-schema)=
##### Schema

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be a valid path type. |
| `"format"` | string | yes | Value MUST be a valid path format. See below. |
| `"href"` | string | yes | Value MUST be a string containing a path in the declared `format`. See below. |

(rfcs:rfc8:path-interface-field-type)=
##### Field: `type`

The `type` field of a `Path` object defines how the path is interpreted. This RFC defines two unprefixed path types: `zarr` and `json`:

- The `"zarr"` type is used for paths that reference nodes in a Zarr array or
  group. Implementations MUST append `zarr.json` to the path to access the
  metadata of the referenced node.
- The `"json"` type is used for paths that reference nodes in a JSON file.

The `type` field of a `Path` object is an extension point. For detail on how to extend the `type` field with new values, see [Extensions](#extensions).

(rfcs:rfc8:path-interface-field-format)=
##### Field: `format`

The `format` field of a `Path` object declares how the `href` string is to be interpreted.
Implementations MUST use the `format` field to interpret the `href` and MUST NOT infer the format from the `href` string itself (e.g. from a `file://` or `https://` prefix).

The following path formats are defined:

| Format | Defined in | Description |
| - | - | - |
| `"relative"` | this RFC | A path relative to the metadata document that contains the `Path` object. |

Only `relative` is a core path format of this RFC.
Implementations that do not support the declared `format` MUST NOT attempt to resolve the path and SHOULD treat the referenced node as unavailable.

Future RFCs may propose additional path formats, such as S3 URLs or chained paths (e.g. for referencing files within a zip file).

(rfcs:rfc8:path-interface-field-href)=
##### Field: `href`

The `href` string locates the referenced document in the declared `format`.
This RFC defines the rules for `href` values with the `relative` format only.

- Relative paths are resolved from the location of the metadata document that contains the `Path` object,
  i.e. from the Zarr group whose `zarr.json` contains it, or from the directory of the standalone JSON file that contains it.
- Relative paths follow the relative path notation defined in [IETF RFC1808](https://datatracker.ietf.org/doc/html/rfc1808).
  Briefly, `.` and `..` are used to navigate the hierarchy and the hierarchy is separated by `/`.
- Relative paths may be used for data stored on traditional file systems as well as other storage protocols, such as HTTP or S3.
  The storage protocol is determined by how the containing document was accessed, not by the `href`.

Examples:
- `./image.ome.zarr`
- `../image.ome.zarr`
- `./registered/em.ome.zarr`
- `s0`

See the [Security](#security) section for guidance on access restrictions.

(rfcs:rfc8:path-interface-resolution)=
##### Path resolution

Resolving a node's `path` means loading the document it locates, and, if the root node of that document has child nodes with a `path`, resolving those in turn.
Resolving node paths MUST NOT form a cycle: a node's `path` MUST NOT resolve to a document that contains that node, directly or through a chain of other node paths.
Implementations MUST detect cycles and MUST stop resolving and report an error when they encounter one.

This rule applies to the `path` fields of nodes.
The `path` field of a [`Reference`](#reference-interface) only locates the document that contains the referenced object and MAY point back to an enclosing document, e.g. from a resolution level to the coordinate systems of its multiscale.

(rfcs:rfc8:declaration-interface)=
#### `Declaration` interface

Every object that can be referenced is declared exactly once.
A declaration is the place where an object's `id` and its other fields are defined, e.g. an entry in a `nodes` or `coordinateSystems` array.

Every declaration MUST have the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be the kind of the declared object, e.g. `"collection"`, `"multiscale"` or `"singlescale"` for nodes and `"coordinate_system"` for coordinate systems. |
| `"id"` | string | yes | Value MUST be the identifier of the declared object. IDs MUST be unique within the JSON document. |

Additional fields, traits and child nodes of an object are defined on its declaration.
Everywhere else, the object is referred to with a [`Reference`](#reference-interface).
Declarations use the `id` key and references use the `ref` key, so that a declaration can always be told apart from a reference.

(rfcs:rfc8:reference-interface)=
#### `Reference` interface

The `Reference` interface is a consistent system for referring to local and remote OME-Zarr metadata objects.
A reference points to an object that is [declared](#declaration-interface) elsewhere, e.g. in the `input` and `output` fields of coordinate transformations.

(rfcs:rfc8:reference-interface-example)=
##### Example

A reference to a coordinate system that is declared in the same document:

```jsonc
"output": { "type": "coordinate_system", "ref": "physical" }
```

A reference to a coordinate system that is declared in another document:

```jsonc
"input": {
  "type": "coordinate_system",
  "path": { "type": "zarr", "format": "relative", "href": "./raw" },
  "ref": "physical"
}
```

(rfcs:rfc8:reference-interface-schema)=
##### Schema

A reference MUST be an object with the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be the `type` of the referenced declaration. |
| `"ref"` | string | yes | Value MUST be the `id` of the referenced declaration. |
| `"path"` | object | no | Value MUST be a `Path` object locating the document that contains the referenced declaration. |

A reference MUST NOT contain any other fields.
In particular, a reference MUST NOT carry traits or child nodes; these belong to the declaration.

(rfcs:rfc8:reference-interface-resolution)=
##### Resolution

- If `path` is absent, `ref` is resolved within the current metadata document.
- If `path` is present, `ref` is resolved within the metadata document that `path` locates.
- The resolved declaration MUST have the `type` that the reference names.
  Implementations MUST report an error if no declaration with the given `ref` exists in the resolved document, or if its `type` does not match.

(rfcs:rfc8:abstract-structure)=
### Abstract structure

The proposal introduces a common Node structure for different types of OME-Zarr
metadata objects. Nodes can represent images or collections, and can be nested
or referenced by path. Collections provide a mechanism for grouping nodes and
may have metadata attached to them, while nodes may carry additional metadata
describing their role within a collection. Nodes can be referenced locally or
remotely, without relying on a file system hierarchy. Images may be added as
nodes to multiple collections. Arbitrary user or implementation metadata may be
added to collections or nodes, which is an opportunity to add metadata that is
only valid for a node in the context of a collection (e.g. rendering settings).

(rfcs:rfc8:abstract-structure-node)=
#### `Node` interface

This RFC defines a basic interface for an OME-Zarr metadata object, which we
name `Node`. The [Node interface](#node) is a consistent JSON structure for
several different types of OME-Zarr metadata object and the root only stores
information used for identifying and referencing the object.

Nodes conform to the [`Declaration` interface](#declaration-interface) and therefore require the `type` and `id` fields.

(rfcs:rfc8:abstract-structure-node-schema)=
##### Schema

Objects that implement `Node` have the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be a string identifying the node type. |
| `"id"` | string | yes | Value MUST be a UUID, MUST be unique within the JSON document, and SHOULD be a UUIDv7+. |
| `"name"` | string | no | Value MUST be a non-empty string intended for human-readable display. |
| `"traits"` | array | no | Value MUST be an array of `Trait` objects. [See traits section](#traits) |

(rfcs:rfc8:abstract-structure-node-field-type)=

##### Field: `type`

The `type` field of a `Node` defines its structure and semantics, including any additional fields it might have.
This RFC defines three node types: `collection`, `multiscale`, and `singlescale`.

The `type` field of a `Node` is an extension point. For detail on how to extend the `type` field with new values, see [Extensions](#extensions).

(rfcs:rfc8:abstract-structure-node-field-version)=
##### Field: `version`

A `Node` object may be used as the root object of the `ome` key, in which case a `version` field, as defined in previous spec versions, is also required.
Non-root `Node` objects SHOULD NOT have a `version` field and MUST NOT have a different `version` value than the root `Node`.

(rfcs:rfc8:abstract-structure-traits)=
#### Traits

Each `Node` has a `traits` field that accepts a list of `Trait` objects. A trait is an object that can be used to enrich any `Node` with arbitrary metadata. A primary use case for traits is the specialization of collections and nodes.

Each trait has minimally a `type` field, which dictates how the trait should be interpreted and defines the other fields it may contain.

This RFC defines one type of trait: [`scene`](#scene-trait).

Trait types are also an extension point, so any extension can define its own traits following the rules set forth in the [“Extensions” section](#extensions).

#### Metadata storage

##### OME-Zarr group

Node metadata may be stored in the `ome` key of the `attributes` container in a `zarr.json` file of a Zarr group.
This is particularly useful for defining the nodes that are stored within a Zarr group. However, there is no limitation to only reference nodes within the Zarr group.

```jsonc
{
  // zarr.json
    "zarr_format": 3,
    "node_type": "group",
    "attributes": {
        "ome": {
            "version": "0.x",
            // Our `collection`-typed node attributes
            // are listed here
            "type": "collection",
            "id": "01a1354d-7cd7-724f-96d6-29c9d60bbf28",
            "name": "zarr.json-example",
            "nodes": [...]
        }
    }
}
```

##### Standalone JSON

Node metadata may also be stored in standalone JSON files that are stored in arbitrary locations and have a file name ending in `.json`.
Here, the metadata is stored in the `ome` key of the root object.
Standalone files are useful for persisting groupings of images that may or may not be stored in the same folder hierarchy.

```jsonc
{
  // an arbitrary json
    "ome": {
        "version": "0.x",
        // Our `collection`-typed node attributes
        // are listed here
        "type": "collection",
        "id": "01a1354d-7cd7-7658-95d8-4f1e9cadc9c6",
        "name": "standalone-example",
        "nodes": [...]
    }
}
```

(rfcs:rfc8:metadata-storage-consolidated)=
##### Consolidated metadata (non-normative)

This section is non-normative.

Resolving a tree of path-referenced nodes requires fetching the `zarr.json` of each referenced Zarr group (see [Performance](#performance)).
To reduce the number of requests, writers MAY use [Zarr consolidated metadata](https://github.com/zarr-developers/zarr-specs/pull/309) to bundle the `zarr.json` documents of referenced nodes into the `zarr.json` of the root group.
Consolidated metadata stores a copy of the `zarr.json` of every group and array below the root group as a flat mapping, keyed by the path relative to the root group.
A reader that supports consolidated metadata can then load the whole tree in a single request.
This also applies to nested collections: their `zarr.json` documents may be consolidated into the root, while they remain separate documents with their own `ome` metadata.

```jsonc
{
  // zarr.json of the root group
    "zarr_format": 3,
    "node_type": "group",
    "attributes": {
        "ome": {
            "version": "0.x",
            "type": "collection",
            "id": "01a1354d-7cd7-724f-96d6-29c9d60bbf28",
            "name": "consolidated-example",
            "nodes": [
                {
                    "type": "multiscale",
                    "id": "01a1354d-7cd7-7a1e-8f3c-2b6d4e5f7a81",
                    "name": "image",
                    "path": { "type": "zarr", "format": "relative", "href": "./image" }
                }
            ]
        }
    },
    "consolidated_metadata": {
        "kind": "inline",
        "must_understand": false,
        "metadata": {
            "image": {
                // copy of ./image/zarr.json
                "zarr_format": 3,
                "node_type": "group",
                "attributes": { "ome": { ... } }
            },
            "image/s0": {
                // copy of ./image/s0/zarr.json
                "zarr_format": 3,
                "node_type": "array",
                ...
            }
        }
    }
}
```

The individual `zarr.json` files remain the source of truth.
The consolidated copy is only a cache, and it can go stale when a referenced node is modified without updating the root.
Readers that support consolidated metadata use the consolidated copy when it is present, so they see outdated metadata until the root is consolidated again.
Keeping the cache consistent with the individual files would require an external transaction system, such as [Icechunk](https://icechunk.io) or a custom database.

Consolidated metadata is a pure Zarr mechanism, and this RFC adds no OME-specific rules to it.
Readers that do not support consolidated metadata fetch the `zarr.json` of each referenced node as usual.
Consolidated metadata can only cover Zarr nodes within the hierarchy of the root group; standalone JSON files and nodes outside the root group are always fetched individually.

At the time of writing, the Zarr-level specification of consolidated metadata is incomplete, which limits what this RFC can say about it.
A future revision may give more guidance once the Zarr specification is finalized.

### New and modified core classes

Building on the core building blocks and the abstract structure, the following
sections define concrete node types for collections, single-scale images, and
multiscale images, and integrate coordinate systems and transformations from
RFC-5. Existing OME-Zarr structures such as multiscales are reworked within the Node/Collection-based framework.

(rfcs:rfc8:collection-node)=
#### `Collection` node

[Collections](#collection-node) are arbitrary groups of `Node`s` which can be
specialised for different use cases. Collections can be used to group together
images, including segmentations, prediction maps and other derived images as
well as other data types ("nodes"). Collections can be nested. Collections can
have metadata attached. Within collections, nodes can also have metadata, which
complements or overrides the nodes' own metadata. Nodes within collections are
referenced by paths instead of relying on a file system hierarchy.

(rfcs:rfc8:collection-node-schema)=
##### Schema

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be `"collection"`. |
| `"id"` | string | yes | Value MUST be a UUID, MUST be unique within the JSON document, and SHOULD be a UUIDv7+. |
| `"name"` | string | no | Value MUST be a non-empty string intended for human-readable display. |
| `"nodes"` | array | no | Value MUST be an array of `Node` objects. |
| `"path"` | object | no | Value MUST be a `Path` object. |
| `"traits"` | array | no | Value MUST be an array of `Trait` objects. [See traits section](#traits). |

Either `"nodes"` or `"path"` MUST be present, but not both.

If `"path"` is present, the document it resolves to MUST have a root node of type `collection`.

```jsonc
{
  "ome": {
      "version": "0.x",
      "type": "collection",
      "id": "01a13543-f211-74ac-b6d8-da304a733e68",
      "name": "proj_gallery",
      "nodes": [{
          "type": "collection",
          "name": "gallery1",
          "path": {
            "type": "json",
            "format": "relative",
            "href": "../gallery.json"
          }
      }, ...]
  }
}
```

(rfcs:rfc8:singlescale-node)=
#### `Singlescale` node

A `Singlescale` node represents one resolution level of an OME-Zarr multiscale image.
This new interface replaces the dataset metadata defined in the previous versions of the OME-Zarr specification.

(rfcs:rfc8:singlescale-node-schema)=
##### Schema

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be `"singlescale"`. |
| `"id"` | string | yes | Value MUST be a UUID, MUST be unique within the JSON document, and SHOULD be a UUIDv7+. |
| `"name"` | string | no | Value MUST be a non-empty string intended for human-readable display. |
| `"path"` | object | no | Value MUST be a `Path` object. |
| `"attributes"` | object | yes | Value MUST be a dictionary. [See attributes section](#attributes). Required because it MUST contain `coordinateTransformations`.|

`Singlescale` nodes represent resolution levels within a multiscale pyramid.

(rfcs:rfc8:singlescale-node-field-coordinateTransformations)=
##### Field: `coordinateTransformations`

`Singlescale` nodes MUST have a `coordinateTransformations` key in their `attributes`, which
- is an array of transformation objects, that conform to the [coordinate transformations](#coordinate-transformations) specification
- contain and only contain a single `scale` transformation, or a `sequence` of a `scale` transformation followed by a `translation` transformation.
- The `input` field of these transformations references the `id` of the  `Singlescale` node itself.
- The `output` field references the `id` of a coordinate system defined under `coordinateSystems` in a `Multiscale` node.

```jsonc
{
  "ome": {
    "version": "0.x",
    "type": "singlescale",
    "id": "01a13da8-7b6e-7030-ac88-b8013ef53020",
    "name": "s0",
    "path": {
      "type": "zarr",
      "format": "relative",
      "href": "./s0"
    },
    "attributes": {
      ...
    }
  }
}
```

(rfcs:rfc8:multiscale-node)=
#### `Multiscale` node

A `Multiscale` node represents an OME-Zarr multiscale image.
This new interface replaces the multiscale metadata defined in the previous versions of the OME-Zarr specification.

(rfcs:rfc8:multiscale-node-schema)=
##### Schema

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be `"multiscale"`. |
| `"id"` | string | yes | Value MUST be a UUID, MUST be unique within the JSON document, and SHOULD be a UUIDv7+. |
| `"name"` | string | no | Value MUST be a non-empty string intended for human-readable display. |
| `"nodes"` | array | no | Value MUST be an array of `Singlescale` objects. |
| `"path"` | object | no | Value MUST be a `Path` object. |
| `"traits"` | object | yes | Value MUST be an array of `Trait` objects. [See traits section](#traits).|

Either `"nodes"` or `"path"` MUST be present, but not both.

If `"path"` is present, the document it resolves to MUST have a root node of type `multiscale`.

```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "multiscale",
        "id": "01a1354c-9611-72bb-b440-6db340b0f0bd",
        "name": "imagepyramid1",
        "nodes": [
          {
            "id": "01a13daa-329c-7242-9610-af256d94c029",
            "name": "s0",
            "type": "singlescale",
            "path": {
              "type": "zarr",
              "format": "relative",
              "href": "./s0"
            },
            "traits": [
              ...
            ]
          },
          ...
        ]
    }
}
```

#### Coordinate transformations

Coordinate systems and transformations can be stored in two distinct locations:

- For single multiscale or singlescale images, they can be stored in the `ome` key of the `attributes` container in a `zarr.json` file of the multiscales zarr group.
- For collections of two or more images in a common coordinate system, RFC-5 defines a parent-level metadata format.
  In this layout, `coordinateTransformations` define relationships between different `coordinateSystems`,
  which may be associated to multiscale images:

```jsonc
{
  "coordinateTransformations": [
    {
      "type": "translation",
      "translation": [0, 0, 100],
      "input": {
        // references coordinate system 'physical' defined in image_1
        "type": "coordinate_system",
        "ref": "01a13dae-bfca-77db-9ee9-7d0de8967ff3",
        "name": "physical",
        "path": {"type": "zarr", "format": "relative", "href": "./image_1"},
      },
      "output": {
        // references coordinate system 'world' defined in the same document
        "ref": "01a13e01-a07b-711a-9443-6b8fe29e8422",
        "name": "world"
      }
    }
  ]
}
```

In a change from RFC-5, coordinate systems are referenced by ID and not by name.
Each `input` and `output` is a typed [`Reference`](#reference-interface) that names the kind of the target (e.g. `coordinate_system`) and its `id` in the `ref` field.

```jsonc
{
  "ome": {
    "version": "0.x",
    "type": "collection",
    "name": "Tiles",
    "id": "01a1354d-7cd7-78c5-bda9-cb72c508924d",
    "traits": [
      {
        "type": "scene",
        "coordinateSystems": [
          {
            "type": "coordinate_system",
            "id": "01a13e01-a07b-711a-9443-6b8fe29e8422",
            "name": "world",
            "axes": [...]
          }
        ],
        "coordinateTransformations": [
          {
            "type": "translation",
            "translation": [0, 0, 100],
            "input": {
              "type": "coordinate_system",
              "ref": "01a13dae-bfca-77db-9ee9-7d0de8967ff3",  //physical
              "path": {
                "type": "zarr",
                "format": "relative",
                "href": "./tile_0.zarr"
              },
            },  // references coordinate system "physical" defined in tile_0
            "output": {
              "type": "coordinate_system",
              "ref": "01a13e01-a07b-711a-9443-6b8fe29e8422"
            }  // references coordinate system "world" defined in same node
          },
          {
            "type": "translation",
            "translation": [100, 0, 0],
            "input": {
              "type": "coordinate_system",
              "ref": "01a13e05-943e-77e2-b23d-190a5c03f019",  // physical
              "path": {
                "type": "zarr",
                "format": "relative",
                "href": "./tile_1.zarr"
              }
            },  // references coordinate system "physical" defined in tile_1
            "output": {
              "type": "coordinate_system",
              "ref": "01a13e01-a07b-711a-9443-6b8fe29e8422"
            }  // references coordinate system "world" defined in same node
          }
        ]
      }
    ],
    "nodes": [
      {
        "type": "multiscale",
        "id": "01a1354d-7cd7-70c1-ba68-99bbccce4122",
        "name": "Tile 0",
        "path": {
          "type": "zarr",
          "format": "relative",
          "href": "./tile_0.zarr"
        }
      },
      {
        "type": "multiscale",
        "id": "01a1354d-7cd7-7dc3-b31d-f5f168a71e87",
        "name": "Tile 1",
        "path": {
          "type": "zarr",
          "format": "relative",
          "href": "./tile_1.zarr"
        }
      }
    ]
  }
}
```

The `type` field of a coordinate transformation defines its mathematical operation. RFC-5 defines several unprefixed transformation types including `identity`, `scale`, `translation`, and others.

The `type` field of a `CoordinateTransformation` is an extension point. For detail on how to extend the `type` field with new values, see [Extensions](#extensions).

##### `Coordinate System` interface

The `Coordinate System` objects have the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be `"coordinate_system"`. |
| `"id"` | string | yes | Value MUST be a UUID, MUST be unique within the JSON document, and SHOULD be a UUIDv7+. |
| `"name"` | string | no | More descriptive name for the coordinate system, if needed. |
| `"axes"` | array of strings | yes | Value MUST be an array of axes, as defined in RFC-5. |

The `type` field of an `Axis` object is an extension point. For detail on how to extend the `type` field with new values, see [Extensions](#extensions).

##### `Coordinate Transformation` interface

The `Coordinate Transformation` objects have the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be a valid coordinate transform type, as defined in RFC-5. |
| `"input"` | object | yes | Value MUST be a typed [`Reference`](#reference-interface) to the input of the transformation. |
| `"output"` | object | yes | Value MUST be a typed [`Reference`](#reference-interface) to the output of the transformation. |

Additional fields MAY be added as required by the transform type.

Depending on the context, the `input` and `output` references point to different kinds of objects:

| Context | `input` | `output` |
| - | - | - |
| **scene** | `{"type": "coordinate_system", path: {..., "href": "./imageA"}, "ref": "01a13e16-c6d6-7057-8a73-7d977c754ea3"}` | `{"type": "coordinate_system", path: {..., "href": "./imageA"}, "ref": "01a13e16-ff87-701f-a77e-b357eca9fd64"}` |
| **Multiscale > attributes > coordinateTransformations** | {"type": "singlescale", "ref": "01a13e17-4776-72aa-8a9a-f1101d555fc2"} | { "type": "coordinate_system", "ref": "01a13e17-9425-7457-b728-9b545e3bf661"} |

**Multiscale > attributes > coordinateTransformations**: In the context of multiscales transformations, the following requirements apply:
- The `input` field MUST be a reference of type `singlescale` to a [singlescale node](#singlescale-node).
- The `path` field MAY be omitted.
- The `output` field MUST be a reference of type `coordinate_system`.
- The `output` fields of all transformations MUST reference the same coordinate system.

**Node/Multiscales > attributes > scene > coordinateTransformations**:
In the context of node-level transformations between different multiscale collections, the following requirements apply:
- The `input` and `output` fields MUST be references of type `coordinate_system`.
- If the referenced coordinate system is declared in the same metadata document, the `path` field MAY be omitted.
- If the referenced coordinate system is declared in a different metadata document, the `path` field MUST be present.


##### `Scene` trait

The `scene` trait MUST be an object with the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"coordinateSystems"` | array | no | Values MUST be valid instances of [`Coordinate System`](#coordinate-system-interface) objects. |
| `"coordinateTransformations"` | array | yes | Values MUST be valid instances of [`Coordinate transformation`](#coordinate-transformation-interface) objects |

A `scene` metadata object can be defined in the `traits` of a collection to enrich the collection with spatial information of the nodes within the collection.
The `scene` field allows to clearly distinguish between the spatial information pertaining to an individual multiscale image (which is stored in the `traits` of the multiscale)
and the spatial information pertaining to the collection of images (which is stored in the `traits` of the collection).

<!--
These "new and modified core classes" could become concrete extensions
under Damien's proposal
-->

## Extensions

This section describes an extensibility mechanism by which anyone can unilateraly extend the OME-Zarr specification and store, under the `ome` attributes of a `zarr.json` file (or any standalone JSON file), additional metadata that were not envisioned by the specification, all the while maintaining interoperability with OME-Zarr applications.

The basic principle is that an extension compliant with this extensibility mechanism can extend the specification by **defining additional, custom types**. An OME-Zarr extension is therefore essentially a set of custom types.

### Rationale for OME-Zarr extensions

We must acknowledge from the very beginning that, by its very nature, OME-Zarr is already _de facto_ extensible, even before the word “extensibility” makes it into the OME-Zarr specification. Anyone can craft an OME-Zarr fileset that contains data beyond what is described in the current version of the specification. Most if not all OME-Zarr readers are inherently “permissive”, in that they will silently ignore anything that they don’t recognise or don’t expect within an OME-Zarr fileset, so that the presence of additional data will typically not result in an unreadable fileset.

However, if left unchecked, this _de facto_ extensibility could quickly become a barrier to interoperability. If every single creator of OME-Zarr filesets can singlehandedly come up with its own way of representing a particular piece of data, then there is a risk that a particular piece of data may be understandable only by the very same implementation that created it; or (arguably even worse) that a particular piece of data may be understood _differently_ by different implementations. Ultimately, OME-Zarr could end up as a file format that is interoperable in name only: everybody would be using OME-Zarr, but all in slightly incompatible ways.

The solution to that risk is _not_ to “close down” OME-Zarr, to mandate implementations to be non-permissive and to consider that anything that is not explicitly allowed by the specification is forbidden. In fact, even if we wanted to do that, we would have little to no leverage to actually enforce such a policy. Instead, we believe the proper solution is for OME-Zarr to provide a built-in extensibility mechanism. There will be no _obligation_ for people willing to extend OME-Zarr to use the mechanism defined here (again, we would have very little leverage to enforce such an obligation anyway), but (voluntarily) using it would allow the OME-Zarr community to _manage_ the interoperability issues.

The manageability aspect is important because an extensibility system alone cannot prevent (a) the risk of a fragmented ecosystem, with an uncontrolled proliferation of redundant extensions; (b) the risk of uninterpretable extensions, that cannot be understood beyond their originating implementations. Instead, the OME-Zarr community will need to take steps to actively monitor the extension ecosystem and intervene as needed to try avoiding redundant or concurrent extensions in favour of promoting cooperation.

In short, we propose that OME-Zarr should offer a system of _manageable_ extensibility, which would be preferred over the _de facto_ extensibility that is already offered by the very nature of OME-Zarr.

This proposal requires that extensions be treated as first-class objects within the OME-Zarr specification, which is the purpose of this section of the present RFC.

In the rest of this document, the word “extension” will be understood as referring strictly to “an extension compliant with the proposed extensibility mechanism”, ignoring all other possibilities of extensions.

#### Desired characteristics for OME-Zarr’s extensibility mechanism

(Those characteristics are _not_ listed in any particular order.)

1. Anyone can define an extension. Prior permission is not required.
2. An extension can extend another extension, without arbitrary limits as to the “depth” of an extension chain.
3. Data from an extension must be preserved by all implementations – including implementations that are not explicitly aware of that particular extension. OME-Zarr readers must be able to pass unknown extensions’ data down to their client code.
4. Extensions must be unambiguously identifiable.
5. An OME-Zarr reader must be able to quickly determine (and report to client code) the extensions that are being used within an OME-Zarr fileset, if any.
6. OME-Zarr implementations must be able to unambiguously state which extensions they explicitly support, preferably in machine-readable form.
7. Any extension must be clearly documented. The documentation must state what additional data the extension is intended to store, the purpose of such data, and where to find them.
8. Any extension must provide a formal schema describing the shape of its data and allowing even an implementation that does not explicitly support the extension to perform basic validation of the extension’s data.
9. An extension should have a way to self-declare how “important” the extension is for a correct understanding of the OME-Zarr fileset in which it is used.
10. The extensibility mechanism must ensure the absence of naming conflicts between two extensions developed independently.
11. There must be a reasonably straightforward way to “promote” an extension so that it becomes part of the next version of the OME-Zarr specification, if there is agreement among the OME-Zarr community (as reached, most likely, through the existing RFC process) about the usefulness of the extension.
12. The promotion of an extension must not invalidate existing data created before the extension was promoted.

### Extension and custom type identifiers

An extension MUST have a unique identifier, which SHOULD be an absolute URI that SHOULD end with either `#` or `/`.

All custom types defined by an extension MUST have their own unique identifier, which MUST be built by appending a type-specific suffix (the type’s _local name_, within the context of the extension that defines it) to the extension’s identifier.

> For example, if an extension with the identifier `https://example.org/my-ngff-extension/` defined a new type of coordinate transformation named `NonLinearTransform`, the identifier for the new type would be `https://example.org/my-ngff-extension/NonLinearTransform`, with `NonLinearTransform` being the type’s local name.

### Extension points

Custom types can be used in lieu of existing types defined by the specification, at defined extension points.

An extension point is a `type` field within an object where (1) the value of the `type` field dictates how the rest of the object is to be interpreted, and (2) the OME-Zarr specification explicitly states that the allowed values in the `type` field “MAY be extended” (exact phrasing does not matter as long as the intention is clear).

At such an extension point, a custom type may be used by specifying its identifier in the `type` field, in lieu of one the values allowed by the specification. The custom type, as defined by the extension it belongs to, will then dictate how the object is to be interpreted.

For example, the [`Path` interface](#path-interface) proposed by this very RFC has an extension point: it has a `type` field defined as accepting two values (`zarr` and `json`); the value dictates how the entire `Path` object is to be interpreted; and the specification explicitly states it is an extension point. An extension could then define a new, custom type of `Path` object.

### Extension manifest

Any JSON file that makes use of extension-defined custom types MUST declare an extension manifest, which MUST list every extension used within the file. The manifest is stored under the `extensionManifest` key of the `ome` object.

The extension manifest has several purposes, all stemming from the intention of making extensions “manageable”. Notably, it is intended to:

- allow figuring out at a glance (1) whether an OME-Zarr image is making use of extensions at all, and (2) if so, which extensions are being used;
- provide informations about an extension, including pointers to human-readable documentation and machine-readable schemas.

When the metadata of an OME-Zarr image are spread over several JSON files in different Zarr groups, each file MUST carry its own manifest, listing the extensions used in that file. Additionally, the manifest in the root-level `zarr.json` file SHOULD list every extension used in any of the downstream files, so that a reader can find out which extensions are being used in an entire image without having to check all the JSON files in the Zarr hierarchy.

The manifest MUST be an array of extension objects. Each listed extension MUST be an object with the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"id"` | string | yes | Identifier for the extension. SHOULD be an absolute URI. MUST be unique within the manifest. |
| `"version"` | string | yes | Version of the extension used in the document. |
| `"name"` | string | no | Short human-readable name, for display. |
| `"description"` | string | no | One-sentence human-readable description of the extension. |
| `"mustUnderstand"` | boolean | no | Whether knowledge of this extension is required for faithful processing of the data. Default `false`. |
| `"homepage"` | string | no | URL to a homepage or documentation page for the extension. |
| `"schemas"` | list[Schema] | yes | Formal schemas describing the custom types defined by the extension. |

Each item in the `schemas` list MUST be an object with the following fields:

| Field | Type | Required? | Notes |
| - | - | - | - |
| `"type"` | string | yes | Value MUST be either `linkml` or `jsonschema`. |
| `"url"` | string | yes | A URL pointing to the schema. |

Future versions of this specification may add new types of schemas.

#### Use of schemas

An extension MUST provide at least one schema in one of the supported schema definition languages (currently `linkml` or `jsonschema`).

When using a JSON-Schema, every custom object type defined by the extension MUST be formally described in an entry within the `$defs` key of the JSON-Schema, where the name of the entry MUST match the custom type’s local name.

When using a LinkML schema, every custom object type defined by the extension MUST be formally described by a LinkML class definition whose name MUST match the custom type’s local name.

Readers that do not explicitly support an extension MAY use the provided schema(s) to attempt to validate the extension’s data. If they do so, they SHOULD treat any validation error the same way as they would treat any non-compliant data.

Viewers that do not explicitly support an extension MAY use the provided schema(s) to display the extension’s data in a meaningful way, for example by using any provided human-readable labels and descriptions for the objects and fields that make up the extension’s data.

#### The `mustUnderstand` flag

Some extensions are intended to only add non-essential information, such as a viewer’s display settings, or a description of how the image was acquired. A reader that ignores such extensions can still interpret the rest of the data correctly. Other extensions may change what the data _means_, e.g. a coordinate transformation the reader cannot apply, or pixel values that are not intensities. A reader that ignores those would be at risk at misleading its users. The `mustUnderstand` flag lets writers state which case applies for any given extension.

When a reader encounters a custom type that (1) it does not explicitly support and (2) is defined by an extension whose `mustUnderstand` flag is set to `true`, the reader MUST NOT silently fall back to a generic interpretation of the object; it MAY skip the object and report it as unsupported; it MAY continue to interpret the rest of the dataset. When the custom type is a [trait](#traits), this paragraph applies to the object that is carrying the trait.

Readers MAY use the extension manifest to decide up front whether an OME-Zarr image is likely to be fully supported, without waiting to encounter an unknown custom type.

Writers SHOULD set `mustUnderstand: true` for a given extension only when ignoring the extension’s data would cause _incorrect_ interpretation, not merely _less complete_ interpretation.

### Examples

#### Extending the `Path` object

This example illustrates an extension with ID `https://ngff.example.org/omero/` that adds a a custom type of `Path` that points to an image on an OMERO server rather than to a file.

```jsonc
{
    "ome": {
        "version": "0.x",
        "extensionManifest": [
            {
                "id": "https://ngff.example.org/omero/",
                "version": "0.3",
                "name": "OMERO connector extension for OME-Zarr",
                "description": "An extension to allow referring to remote OMERO images from within a OME-Zarr image.",
                "homepage": "https://ngff.example.org/omero/",
                "schemas": [
                    {
                        "type": "jsonschema",
                        "url": "https://schemas.ngff.example.org/omero/0.3/omero.json"
                    }
                ]
            }
        ],
        "type": "collection",
        "nodes": [
            {
                "name": "original-acquisition",
                "type": "multiscale",
                "path": {
                    "type": "https://ngff.example.org/omero/OMEROPath",
                    "href": "https://omero.example.org/webclient/?show=image-48213",
                    "host": "omero.example.org",
                    "port": 4064,
                    "imageId": 48213
                }
            },
            {
                "name": "segmentation",
                "type": "multiscale",
                "path": {"type": "zarr", "format": "relative", "href": "./segmentation"}
            }
        ]
    }
}
```

The JSON-Schema for that extension could be:

```jsonc
{
    "$defs": {
        "OMEROPath": {
            "description": "A Path that points to an image on an OMERO server.",
            "properties": {
                "type": {
                    "description": "The object's type",
                    "type": "string"
                },
                "href": {
                    "description": "Full URL to the image.",
                    "type": "string"
                },
                "host": {
                    "description": "The hostname of the OMERO server.",
                    "type": "string"
                },
                "port": {
                    "description": "The TCP port to use when connecting to the server.",
                    "type": "number",
                    "default": 4064
                },
                "imageId": {
                    "description": "The ID of the image on the OMERO server.",
                    "type": "number"
                }
            },
            "required": [
                "type",
                "href",
                "host",
                "imageId"
            ],
            "type": "object"
        }
    }
}
```

A LinkML schema for the same extension, if one were provided, could be:

```yaml
id: https://ngff.example.org/omero/
name: OMERO connector extension for OME-Zarr
prefixes:
    linkml: https://w3id.org/linkml/
imports:
    - linkml:types
default_range: string
classes:
    OMEROPath:
        description: A Path that points to an image on an OMERO server.
        attributes:
            type:
                description: The object’s type.
                required: true
            href: 
                description: Full URL to the image.
                range: uri
                required: true
            host:
                description: The hostname of the OMERO server.
                required: true
            port:
                description: The TCP port to use when connecting to the server.
                range: integer
                ifabsent: int(4064)
            imageId:
                description: The ID of the image on the OMERO server.
                range: integer
                required: true
```

#### A custom `coordinateTransformation`

In this example, a `https://ngff.example.com/transforms/` extension defines a custom type of coordinate transformation that uses the “thin plate spline” technique.

```jsonc
{
    "ome": {
        "version": "0.x",
        "extensionManifest": [
            {
                "id": "https://ngff.example.com/transforms/",
                "version": "0.1",
                "name": "OME-Zarr additional transforms",
                "description": "An extension providing additional coordinate transformations.",
                "homepage": "https://example.com/ngff-extensions/transforms/",
                "schemas": [
                    {
                        "type": "jsonschema",
                        "url": "https://schemas.example.org/transforms/0.1/transforms.json"
                    }
                ]
            }
        ],
        "type": "collection",
        "traits": {
            "scene": {
                "coordinateSystems": [
                    {
                        "id": "physical",
                        "axes": [
                            {"name": "y", "type": "space", "units": "micrometer"},
                            {"name": "x", "type": "space", "units": "micrometer"}
                        ]
                    },
                    {
                        "id": "registered",
                        "axes": [
                            {"name": "y", "type": "space", "units": "micrometer"},
                            {"name": "x", "type": "space", "units": "micrometer"}
                        ]
                    }
                ],
                "coordinateTransformations": [
                    {
                        "type": "https://ngff.example.com/transforms/ThinPlateSpline",
                        "input": {"id": "physical"},
                        "output": {"id": "registered"},
                        "controlPoints": {"type": "zarr", "format": "relative", "href": "./tps/control_points"},
                        "regularization": 0.01
                ]
            },
        "nodes": [ /* ... */ ]
    }
}
```

#### Viewer rendering states

In this example, two OME-Zarr viewers, “Viewer A” and “Viewer B” each have their own extension providing a custom trait to store their own display settings for the same image.

```jsonc
{
    "ome": {
        "version": "0.x",
        "extensionManifest": [
            {
                "id": "https://viewer-a.example.com/settings/",
                "version": "0.1",
                "name": "Viewer-A visualisation settings",
                "description": "An extension to store Viewer A-specific settings into an OME-Zarr image",
                "homepage": "https://viewer-a.example.com/",
                "schemas": [
                    {
                        "type": "jsonschema",
                        "url": "https://viewer-a.example.com/ngff/schemas/settings-0.1.json"
                    }
                ]
            },
            {
                "id": "https://viewer-b.example.org/rendering/",
                "version": "0.2",
                "name": "Viewer-B visualisation settings",
                "description": "An extension to store Viewer B-specific settings into an OME-Zarr image",
                "homepage": "https://viewer-b.example.org/",
                "schemas": [
                    {
                        "type": "jsonschema",
                        "url": "https://viewer-b.example.org/ngff-embedded-settings.json"
                    }
                ]
            }
        ],
        "type": "collection",
        "nodes": [
            {
                "name": "nuclei",
                "type": "multiscale",
                "path": {"type": "zarr", "format": "relative", "href": "./nuclei"},
                "traits": [
                    {
                        "type": "https://viewer-a.example.com/settings/Layer",
                        "visible": true,
                        "opacity": 1.0,
                        "shader": "void main() { emitRGB(vec3(0.0, 0.0, toNormalized(getDataValue()))); }"                      
                    },
                    {
                        "type": "https://viewer-b.example.org/rendering/Display",
                        "colormap": "blue",
                        "contrastLimits": [110, 3800],
                        "gamma": 1.0,
                        "blending": "additive"
                    }
                ]
            }
        ]
    }
}
```

## Type tiers

All object types that may be found in a OME-Zarr JSON file ultimately belong to one of two categories: **built-in types**, which are defined by the OME-Zarr specification, and **custom types**, which are defined by [extensions](#extensions).

Built-in types are identified by a specification-defined keyword, while custom types are identified by an identifier derived from the identifier of the extension that defines them, as explained in the [“Extensions” section](#extension-and-custom-type-identifiers).

Built-in types can further be subdivided into **core types**, **recommended types**, and **optional**. As the name suggests, the difference between the two categories lies in what is required from OME-Zarr implementations: a compliant OME-Zarr implementation MUST support all core types, SHOULD support recommended types, and MAY support optional types.

Custom types are initially all OPTIONAL. However, future revisions of the specification may promote a particular custom type to the level of a recommended type, if there is agreement within the community (through the specification revision process, that is RFCs) that the custom type is well designed, useful, and widespread enough to warrant such a promotion.

The following table recapitulates the currently defined built-in types along with their assigned tier.

| Type name | Type of… | Tier | Origin |
| - | - | - | - |
| `collection` | Node | Core type | This RFC |
| `multiscale` | Node | Core type | This RFC |
| `zarr` | Path | Core type | This RFC |
| `json` | Path | Core type | This RFC |
| `scene` | Trait | Core type | This RFC |
| `array` | Axe | Core type | RFC-5 |
| `space` | Axe | Core type | RFC-5 |
| `time` | Axe | Core type | RFC-5 |
| `channel` | Axe | Core type | RFC-5 |
| `coordinate` | Axe | Core type | RFC-5 |
| `identity` | Coordinate transformation | Core type | RFC-5 |
| `scale` | Coordinate transformation | Core type | RFC-5 |
| `translation` | Coordinate transformation | Core type | RFC-5 |
| `mapAxis` | Coordinate transformation | Recommended type | RFC-5 |
| `affine` | Coordinate transformation | Recommended type | RFC-5 |
| `rotation` | Coordinate transformation | Recommended type | RFC-5 |
| `sequence` | Coordinate transformation | Optional | RFC-5 |
| `displacements` | Coordinate transformation | Optional | RFC-5 |
| `coordinates` | Coordinate transformation | Optional | RFC-5 |
| `bijection` | Coordinate transformation | Optional | RFC-5 |
| `byDimension` | Coordinate transformation | Optional | RFC-5 |

Any future RFC that defines its own types MUST explicitly state for each type whether it is intended that implementations MUST support it (categorizing it as a core type), SHOULD support it (categorizing it as a recommended type), or MAY support it (categorizing it as an optional type). A type defined without such an explicit statement is inferred to be an optional type.

## User stories

### 1. Visualize multiple images at once
Several viewers are capable of visualizing multiple images, that can map to a common coordinate space, at once. Examples are Webknossos, Neuroglancer, MoBIE and OMERO.figure. All of these tools have developed their own JSON-based metadata to combine multiple images in a collection, see "[Prior art and references](#prior-art-and-references)". In addition to mere path references of the images, this metadata also contains information about coordinate transforms and rendering settings.

As there is no standard-compliant way in OME-Zarr to describe multiple images in one entity, users need to copy multiple links to interoperably visualize multiple images.

RFC-5 introduced the `scene` metadata, which partially solved this issue.
However, with this proposal we aim to embed it in a more flexible collection mechanism.

### 2. m:n segmentations
While OME-Zarr has support for attaching labels to images, the support is not sufficient for many use cases.
There are multiple label images that can be attached to a single image. This is a 1:n relationship. However, m:n relationships would be desired because labels might be related to multiple images. Examples for that are:
- Multiple correlated images express the same feature that is being labeled.
- Channels are stored in multiple images instead of in the same image.

Additionally, there are other types of derived images, such as prediction maps, which cannot currently be represented by OME-Zarr. In comparison to label maps, where each voxel is assigned a discrete ID, prediction maps have a channel per segmentation class (or similar) and each voxel is assigned a probability or other continuous value.

### 3. Shallow copies of images with segmentations
Many workflow engines operate by taking input images and producing output images. In many cases, it is desired to keep the input images unchanged.
Let's assume the example of a pixel classification task. This task would take an OME-Zarr image as input and produce a prediction map. To express the relationship between input image and output prediction, the task could create a collection that contains the prediction and links to the image (i.e. shallow copy). The output collection could then be used to visualize both at the same time. This is applicable to a wide variety of workflow tasks with the result that the outputs of each task can be visualized or further processed independent of other tasks.

```
├─ input_image.zarr
│  ├─ zarr.json # OME-Zarr multiscale
│  ├─ 0
│  └─ ...
└─ output_collection.zarr
   │  # includes collection metadata and link to "../input_image.zarr"
   ├─ zarr.json
   └─ prediction.zarr
      ├─ zarr.json # OME-Zarr multiscale
      ├─ 0
      └─ ...
```

Examples for such workflow systems:
- [Voxelytics](https://voxelytics.com)
- [Fractal](https://fractal-analytics-platform.github.io/)
- [Nextflow](https://www.nextflow.io/)

### 4. Correlative imaging

Several applications in microscopy and other imaging domains involve the acquisition of images
of the same object from different angles or with different imaging modalities.
Examples of such applications are (among others) the following:
- Correlative light and electron microscopy (CLEM): In this case,
  a sample is examined with both electron and light microscopy,
  both of which feature their own sets of spatial dimensions.
  A set of coordinate transformations is used to map between the different images.
- Multiview lightsheet: For this application,
  lightsheet microscopes acquire multiple views of the same object from different angles.
  A set of coordinate transformations is used to map between the different views.
- Multimodal medical imaging: Different imaging modalities (e.g., CT, MRI, PET, etc),
  are often used either in conjunction or at different timepoints to observe the same object or anatomical structure.

Such applications require the storage of collections of images and their mutual relationships,
the metadata for which has already been defined by RFC-5 (Coordinate Transformations in OME-NGFF).
In the context of RFC-5, images are part of a collection if they share a common coordinate space
that is defined by coordinate systems and coordinate transformations.
Since the relationships between images are already defined
in a graph-like schema as proposed in this RFC,
the transformations metadata can be represented as a specialized collection
with coordinate systems and transformations as attributes of the collection and nodes.
In a way, coordinate transformations and systems simply become a subset of the more general collection concept.


### 5. High Content Screening (HCS) plates

OME-Zarr high content screening plates are a current example of a narrowly defined type of collection.
They allow grouping OME-Zarr images in multiple hierarchy levels: A plate contains wells, which are organized as row folders with column subfolders in each.
Each well folder can contain a number of images.
There is defined metadata about which wells are in a plate and about which images are in a well at the different hierarchy levels, typically with some additional optional metadata like the acquisitions that exist in a plate and which image belongs to which acquisition.

This hierarchy is very useful for typical experiments where researchers imaged a multi-well plate.
Multiple viewers like MoBIE, napari & ViZarr support displaying the different wells arranged in the plate format given the OME-Zarr HCS metadata, thus avoiding the need for tool-specific metadata and showing the benefits of such collection concepts.

The current HCS spec also has its limitations: It has a strict definition of potential metadata fields at the plate and well level.
There are multiple areas where it would be interesting to extend this spec.
There are [ongoing discussions](https://github.com/ome/ngff/pull/137) about whether individual microscope fields of view (ie. well) should be stored as individual OME-Zarr images or as a single OME-Zarr image and how one would represent [different processing intermediates in a plate](https://forum.image.sc/t/how-to-build-hcs-zarrs-with-multiple-image-types-per-fov/119329).
In these contexts, the current HCS spec lacks flexibility to get additional metadata about how images in a well are related and what a viewer should do with them.
For example, depending on whether an OME-Zarr image in a well is an individual field of view of a given acquisition, a second acquisition of the same region in a plate or an image derived from a given processing operation, the optimal viewer default on whether to show or not show multiple images at once will vary.
A flexible metadata field like `attributes` would allow us to better define such image metadata.
A more flexible HCS collection system could also allow to provide advanced metadata on well positions [when wells have different sizes](https://github.com/ome/ome-zarr-py/issues/240) or address other edge-cases in the current HCS configuration.


### 6. Image Archive
Data archives that support deposition and access to OME-Zarr formatted images have two primary use cases for collections of images.
For the first, users submitting data to deposition databases need ways to aggregate collections of images in their data upload structure, and do so in a way that supports describing how those images relate (e.g. parts of the same acquisition series, plate/well data as mentioned above).
This can then be parsed during data submission, and used to create appropriate database records.

Secondly, when providing outgoing access to data, archives want to provide groupings of images that allow compatibility with data exploration and visualisation tools. Given the increasingly rich ecosystem of these tools (mentioned across these use cases, and including grid views, segmentations, multiple images and plate/well data) standardisation is necessary to avoid the need to produce view/exploration schema for each tool.

### 7. Rendering settings
Viewers, such as Webknossos, Neuroglancer, MoBIE and OMERO.figure, are capable of visualizing multiple OME-Zarr images ("layers") in a view.
To share such a view, metadata serialization is required that contains not only links to the images, but also attached metadata of the rendering state.
The rendering state of a collection might contain locations, rotation angles, coordinate systems as well as rendering state of individual layers.
The rendering state of layers might contain pixel transformations (e.g. min/max scaling, colors, shaders), coordinate transformation overrides, visibility settings.

Some of these rendering state attributes might be compatible across implementations, but others might not.
This proposal does not intend to provide a specification for the rendering state itself, but provide metadata containers to store such viewer-specific state.


### 8. Grouping together remote images

When building upon images that have been published by others, it might be useful to create virtual groupings of multiple remotely stored images.
For example, a lab might create automatic segmentations of a large image that has been published by another lab.
While the segmentation would now be published on its own, it could still be published with a link to the original images so that viewers are able to show the segmentation as an overlay on the original data.

### 9. Adding other datatypes to images
When processing images in the OME-Zarr format, a diversity of derived data like segmentation, probability maps, meshes, tables and other formats can be generated.
This proposal does not intend to provide a specification to all these datatypes, but to define the metadata of how related data in Zarr or other formats can be linked to OME-Zarr images.

For  past proposals that have proposed adding datatypes such as [tables](https://github.com/ome/ngff/pull/64) have focused on how tables can be serialised to OME-Zarr.
As these proposals did not proceed to become part of the OME-Zarr spec, different implementers have built their own sub-specs for tables (see e.g. the [ngio table definition](https://fractal-analytics-platform.github.io/ngio/v0.3.2/table_specs/overview/) coming from the Fractal project or the [label table](https://mobie.github.io/specs/mobie_spec.html#table-data) in MoBIE).
While future proposals and extensions may define datatypes like tables more strictly, this proposal offers a general way to make such additional data types discoverable.

### 10. Gallery / grid views

It is useful to visualise similar images in a grid view where all images are visible as "thumbnails", which in the case of OME-Zarr can simply be the lowest resolution version of the data. Like this, users can have an overview of all the data and can then decide to "zoom in" on some datasets to explore them in higher resolution.

Implementations of this concept include:
- [Zarrcade](https://github.com/JaneliaSciComp/zarrcade)
- [BioFile Finder](https://bff.allencell.org/)
- [MoBIE grid views](https://mobie.github.io/tutorials/image_grids_and_tables.html)
- [OME2024 NGFF challenge](https://ome.github.io/ome2024-ngff-challenge/)

For example, [this table](https://docs.google.com/spreadsheets/d/1t5xB0p0zd2-a6ynV-JAuLJqs-mg-pFFikhfmQGZwRpI/edit?usp=sharing) defines a MoBIE grid view of three OpenOrganelle vEM images along with label images of mitochondria segmentation. It can be opened in MoBIE via the "Open Simple Collection Table" menu entry:

![MoBIE grid view](./assets/mobie_grid_view.jpg)

#### Example: A grid view gallery

A gallery view could also be represented within the proposed collection JSON as shown in the below example.

Note that the grid view is modelled here as a collection of collections, where the collection at each grid position includes the raw EM image and the mitochondria segmentation label mask image.

Also note the presence of a MoBIE-specific extension, which adds a MoBIE-specific trait to the collection speciying that the data should be laid out in a grid.

```jsonc
{
    "ome": {
        "version": "0.x",
        "extensionManifest": [
            {
                "id": "https://mobie.example.org/settings/",
                "version": "0.1",
                "name": "MoBIE FIJI viewer visualisation settings",
                "description": "An extension to store MoBIE-specific settings into a OME-Zarr image",
                "homepage": "https://mobie.example.org/",
                "schemas": [
                    {
                        "type": "jsonschema",
                        "url": "https://mobie.example.org/ngff/schemas/settings-0.1.json"
                    }
                ]
            }
        ],
        "type": "collection",
        "name": "openorganelle-mito-gallery",
        "nodes": [
            {
                "name": "jrc_hela-3",
                "type": "collection",
                "nodes": [
                    {
                        "name": "fibsem-uint16",
                        "type": "multiscale",
                        "path": {
                          "type": "zarr",
                          "format": "http",
                          "href": "https://janelia-cosem-datasets.s3.amazonaws.com/jrc_hela-3/jrc_hela-3.zarr/em/fibsem-uint16",
                        }
                    },
                    {
                        "name": "mito_seg",
                        "type": "multiscale",
                        "path": {
                          "type": "zarr",
                          "format": "http",
                          "href": "https://janelia-cosem-datasets.s3.amazonaws.com/jrc_hela-3/jrc_hela-3.zarr/labels/mito_seg",
                        },
                    }
                ]
            },
            {
                "name": "jrc_macrophage-2",
                "type": "collection",
                "nodes": [
                    {
                        "name": "fibsem-uint16",
                        "type": "multiscale",
                        "path": {
                          "type": "zarr",
                          "format": "http",
                          "href": "https://janelia-cosem-datasets.s3.amazonaws.com/jrc_macrophage-2/jrc_macrophage-2.zarr/em/fibsem-uint16"
                        }
                    },
                    {
                        "name": "mito_seg",
                        "type": "multiscale",
                        "path": {
                          "type": "zarr",
                          "format": "http",
                          "href": "https://janelia-cosem-datasets.s3.amazonaws.com/jrc_macrophage-2/jrc_macrophage-2.zarr/labels/mito_seg"
                        },
                    }
                ]
            },
            {
                "name": "jrc_jurkat-1",
                "type": "collection",
                "nodes": [
                    {
                        "name": "fibsem-uint16",
                        "type": "multiscale",
                        "path": {
                          "type": "zarr",
                          "format": "http",
                          "href": "https://janelia-cosem-datasets.s3.amazonaws.com/jrc_jurkat-1/jrc_jurkat-1.zarr/em/fibsem-uint16"
                        }
                    },
                    {
                        "name": "mito_seg",
                        "type": "multiscale",
                        "path": {
                          "type": "zarr",
                          "format": "http",
                          "href": "https://janelia-cosem-datasets.s3.amazonaws.com/jrc_jurkat-1/jrc_jurkat-1.zarr/labels/mito_seg"
                        },
                    }
                ]
            }
        ],
        "traits": [
            {
                "type": "https://mobie.example.org/settings/DisplaySettings",
                "grid": true
            }
        ],
    }
}
```

### 11. Training crops cut from non-OME-Zarr sources

Machine-learning pipelines train on small patches ("crops") cut from published datasets. The sources may be OME-Zarr images, but are often archived in other formats such as TIFF, MRC, HDF5 and N5. A data preparation pipeline downloads the files, cuts the planned crops at several resolution levels, writes them as OME-Zarr images, and finally tensorizes them for training.
For reproducibility, four questions should be answerable from the metadata: how the image was made (software, environment, parameters, time); which file the voxels came from, and whether that file has changed since; where in the source volume each crop sits, at each level; and which parts of the image hold data and what was deliberately omitted.

The first question is answered by a provenance record on the node, such as the [`provenance:` Zarr convention](https://github.com/BioImageTools/zarr-convention-provenance), which lists the steps that made it with their inputs and outputs as paths, URIs or identifiers. The other three are relationships, and a provenance record cannot express them: its references are strings, not references to declared objects with coordinate systems, so nothing ties "the file this image was cut from" to a frame or an extent, and nothing says that the raw image and its label share a frame other than a repeated coordinate-system name.

With this proposal the volume is written as a collection rather than as one image with nested labels. The raw image and its label are sibling multiscales placed in a shared coordinate system by the `scene` trait. Each image names the file it was cut from with a reference to a node that stands for that file; the file's own array index space is a coordinate system of that node, so the crop offset is a transform between two declared frames rather than a convention. The planned crops are nodes with an extent and a reference to the image, not copies of its bytes. The provenance record stays beside `ome` in the node's attributes and names the same nodes by their `id`, so a reader can join how a node was made with what it is to the other nodes.

## Other Examples

The examples below demonstrate combinations of various features in this
proposal. Further example can be found under
https://github.com/normanrz/ngff-rfc8-collection-examples/.

### A multiscale group with a single, inlined resolution level

```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "multiscale",
        "name": "multiscales_example",
        "id": "01a13e2d-61b2-7760-9c2a-1aef3d3eeed6",
        "nodes": [
          {
            "id": "01a13e2d-8ae2-7519-a28f-7b278247ca30",
            "name": "s0",
            "type": "singlescale",
            "path": {
              "type": "zarr",
              "format": "relative",
              "href": "./s0"
            },
            "attributes": {
              "coordinateTransformations": [
                {
                  "type": "scale",
                  "scale": [1, 1, 1],
                  "input": {
                    "type": "singlescale",
                    "ref": "01a13e2d-8ae2-7519-a28f-7b278247ca30"
                  },  // references s0
                  "output": {
                    "type": "coordinate_system",
                    "ref": "01a13e2f-a19f-7735-9802-e48c70b1a089"
                  }  // references 'physical', below
                }
              ]
            }
          }
        ],
        "attributes": {
          "coordinateSystems": [
            {
              "type": "coordinate_system",
              "id": "01a13e2f-a19f-7735-9802-e48c70b1a089",
              "name": "physical",
              "axes": [...]
            }
          ]
        }
    }
}
```

### A multiscale group with a single resolution level

The multiscale group contains the following metadata:

```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "multiscale",
        "name": "multiscales_example",
        "id": "01a13e3b-e9c3-71f4-b779-b8a7caf680cf",
        "nodes": [
          {
            "id": "01a13e3c-13a5-7461-b9ba-3393039199b6",
            "name": "s0",
            "type": "singlescale",
            "path": {
              "type": "zarr",
              "format": "relative",
              "href": "./s0"
            },
          }
        ],
        "attributes": {
          "coordinateSystems": [
            {
              "type": "coordinate_system",
              "id": "01a13e3c-4f9e-760c-87bc-61b13430de55",
              "name": "physical",
              "axes": [...]
            }
          ]
        }
    }
}
```

And the `zarr.json` at the location of the resolution level (`./s0/zarr.json`) contains the following metadata:

```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "singlescale",
        "name": "s0",
        "id": "01a13e3f-d53c-727d-bfc2-93269cae5f3a",
        "attributes": {
          "coordinateTransformations": [
            {
              "type": "scale",
              "scale": [1, 1, 1],
              "input": {
                "type": "singlescale",
                "ref": "01a13e3c-13a5-7461-b9ba-3393039199b6"
              },
              "output": {
                "type": "coordinate_system",
                "ref": "01a13e3c-4f9e-760c-87bc-61b13430de55",
                "path": {
                  "type": "json",
                  "format": "relative",
                  "href": "../zarr.json"
                }  // references "physical" in upstream zarr.json
              }
            }
          ]
        }
    }
}
```

### A collection with a multiscale and a nested collection

```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "collection",
        "name": "jrc_hela-1",
        "nodes": [
          {
            "name": "raw",
            "type": "multiscale",
            "path": {
              "type": "zarr",
              "format": "relative",
              "href": "./raw",
            },
            "attributes": {
                "example-viewer:settings": {
                    "isDisabled": true
                },
                ... // arbitrary user-defined metadata
            },
          },
          {
              "name": "nested_collection",
              "type": "collection",
              "path": {
                "type": "json",
                "format": "relative",
                "href": "./nested_collection.json"
              }
          }, ...
        ],
        "attributes": {
            ...
        }
    }
}
```


### A collection with an inlined multiscale

```jsonc
{
    "ome": {
        "version": "0.x",
        "type": "collection",
        "name": "example",
        "nodes": [
          {
            "name": "raw",
            "id": "01a13e40-df14-7774-aafd-aa1a8defcfdd",
            "type": "multiscale",
            "nodes": [
              {
                "id": "01a13e41-0673-7318-96ca-f56480904ce0",
                "name": "raw_0",
                "type": "singlescale",
                "path": {
                  "type": "zarr",
                  "format": "relative",
                  "href": "./raw/0"
                }
              }
            ],
            "attributes": {
              "coordinateTransformations": [
                {
                  "type": "scale",
                  "scale": [1, 1, 1],
                  "input": {
                    "type": "singlescale",
                    "ref": "01a13e41-0673-7318-96ca-f56480904ce0"
                  },
                  "output": {
                    "type": "coordinate_system",
                    "ref": "01a13e41-6a7e-77e5-aa09-f0017d392a6b"
                  }
                }
              ],
              "coordinateSystems": [
                {
                  "type": "coordinate_system",
                  "id": "01a13e41-6a7e-77e5-aa09-f0017d392a6b",
                  "name": "physical",
                  "axes": [...]
                }
              ]
            }
          }
        ]
    }
}
```

## Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [IETF RFC 2119](https://tools.ietf.org/html/rfc2119)

<!--
For the problem(s) solved by this RFC, what constrains the possible solutions?
List other RFCs, or standards (ISO, etc.) which are applicable.
-->


## Stakeholders

<!--
Who has a stake in whether this RFC is accepted?

* Facilitator: The person appointed to shepherd this RFC through the RFC
  process.
* Reviewers: List people whose vote (+1 or -1) will be taken into consideration
  by the editor when deciding whether this RFC is accepted or rejected. Where
  applicable, also list the area they are expected to focus on. In some cases
  this section may be initially left blank and stakeholder discovery completed
  after an initial round of socialization. Care should be taken to keep the
  number of reviewers manageable, although the exact number will depend on the
  scope of the RFC in question.
* Consulted: List people who should review the RFC, but whose approval is not
  required.
* Socialization: This section may be used to describe how the design was
  socialized before advancing to the "Iterate" stage of the RFC process. For
  example: "This RFC was discussed at a working group meetings from 20xx-20yy"
-->

- Visualization developers
- Data management developers
- Image Archives
- Workflow developers

### Socialization
* [Github issue](https://github.com/ome/ngff/issues/31)
* OME-NGFF community meeting April 2024
    * https://docs.google.com/presentation/d/1ANsNdCchmwWR1grhg5-hSrWH5nyPz2QSFYToF56PVc4/edit#slide=id.g2c639a0285f_0_40
* OME meeting 2024 in Dundee
* I2K 2024 in Milan
* OME-NGFF hackathon Zurich 2024
    * https://hackmd.io/OeY6A-ysQQu_NZuG7a-cXQ
* Volume EM GRC Barcelona 2025
* OME-NGFF community meeting June 2025
* OME-NGFF hackathon Zurich 2025
* OME meeting 2026 in Düsseldorf

## Implementation

This RFC has not been implemented yet.

However, several visualization tools already have support for showing multiple images in the same view, see "Prior art and references".
Adopting this new metadata format is achievable with reasonable effort.

It is also expected that programming libraries will support this new metadata format to construct and inspect collections.

<!--
Many RFCs have an "implementation" section which details how the implementation
will work. This section should explain the rough specification changes. The
goal is to give an idea to reviewers about the subsystems that require change
and the surface area of those changes.

This knowledge can result in recommendations for alternate approaches that
perhaps are idiomatic to the project or result in less packages touched. Or, it
may result in the realization that the proposed solution in this RFC is too
complex given the problem.

For the RFC author, typing out the implementation in a high-level often serves
as "[rubber duck debugging](https://en.wikipedia.org/wiki/Rubber_duck_debugging)" and you can catch a lot of
issues or unknown unknowns prior to writing any real code.
-->


## Drawbacks, risks, alternatives, and unknowns
This RFC adds breaking changes that are neither backwards nor forwards compatible, which presents additional complexity to implementations and users.

Defining a custom format instead of reusing existing validated formats introduces risk of design errors.

### Redundant metadata

This RFC introduces the possibility for redundant metadata.
For example, a collection can contain inlined metadata for multiscale images and the multiscale images can also be specified in standalone metadata.
This redundant metadata can go out of sync.
It is the responsibility of implementations to ensure consistency where required.

For reading, implementations SHOULD parse the metadata as available to the implementation from the user-supplied entry point in the OME-Zarr hierarchy.


### New Multiscale/Singlescale metadata

In an effort to unify the existing and new collection types of OME-Zarr, the Multiscale metadata has been changed to mirror the Collection metadata.
Still, it is a specialized node type because of its prevalence and semantic requirements with regards to its Singlescale children nodes.
The Singlescale metadata has been elevated to a first-class node type.
This also allows specifying images that only contain a single resolution level.

These changes are breaking changes to the core of the OME-Zarr specification.

<!--
* What are the costs of implementing this proposal?
* What known risks exist? What factors may complicate your project? Include:
  security, complexity, compatibility, latency, service immaturity, lack of
  team expertise, etc.
* What other strategies might solve the same problem?
* What questions still need to be resolved, or details iterated upon, to accept
  this proposal? Your answer to this is likely to evolve as the proposal
  evolves.
* What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
* What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
* What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
-->

## Abandoned Ideas

### RO Crate
Instead of defining a custom collection format for OME-Zarr, existing collection formats, such as RO Crate, could be utilized.

Research Object Crate ("RO Crate") is a standard format that utilizes JSON-LD to define collections of research artifacts as well as their relationships and other metadata.
Utilizing RO Crate for OME-Zarr collections would allow building upon an existing standard that has been validated in the field.
Additionally, it would allow to easily integrate with other research artifacts, such as publications, talks, protocols, etc.

RO Crate imposes very strict requirements around its metadata.
Every key in a metadata object needs to be backed by a well-defined schema.
While very precise, this presents burdens for adding user-defined metadata.

All the metadata in RO Crate is normalized and hierarchies or relationships are maintained via IDs.
This is a common approach in database systems design.
However, it leads to metadata files which are very hard to read by humans.
This practically necessitates the use of libraries to inspect the metadata, which contradicts a design guideline of Zarr and OME-Zarr.

This RFC proposes adopting a simpler collection specification for OME-Zarr.
RO Crate can still be used alongside collections or images to define metadata and integrate with other research objects.

### JSONLD

The RFC explored the possibility of using JSON-LD as the basis for the collection metadata format.
JSON-LD is a W3C standard for linked data that extends JSON with a `@context` mechanism for defining vocabularies, making JSON documents interoperable with semantic web tools.

#### Benefits

JSON-LD is a widely used and well-supported standard with broad tooling support.
It provides a standardized type system via `@type`, which maps types to IRIs for global uniqueness and is backed by standard validation tooling.
It has built-in support for defining and referencing objects via `@id`, with JSON-LD processors understanding which properties are references versus literals.
Extensibility is handled through contexts: custom vocabularies can be published at URLs and reused across documents, which would eliminate the need for a custom prefix registry.
Namespace management via prefixes (e.g., `mobie:grid`) clearly distinguishes custom terms from core terms and prevents naming collisions between organizations extending the format.

The planned approach would have replaced the `type` and `id` fields with `@type` and `@id`, and added a `@context` field inside the `ome` object referencing a versioned context URL (e.g., `https://ngff.openmicroscopy.org/0.x/context.jsonld`).
Custom extensions would define their namespace in the context array, removing the need for a central prefix registry.
Internal references (e.g., for `input`, `output`, `source`) would use proper JSON-LD semantics via context definitions, allowing string values to be interpreted as references to `@id` values.

#### Downsides

The core problem with JSON-LD for this RFC is that it is based on IRIs, which makes path-based references difficult.
Standard JSON-LD linking does not support relative paths (e.g., `./image.zarr`) or file system paths.
It also does not support the typed path mechanism—distinguishing between `zarr` and `json` path types—that is a central part of this RFC.
This would force a hybrid approach where the `Path` interface uses JSON-LD syntax (`@type`) but does not conform to JSON-LD semantics.

Such a hybrid approach introduces its own complexity.
Users familiar with JSON-LD would expect standard linking to work, but our custom `Path` mechanism is not standard JSON-LD.
Generic JSON-LD processors would not understand the custom path mechanism and would not be able to fully process the metadata.
The benefit of reusing existing JSON-LD tooling would thus be largely negated.

To keep complexity manageable, we would need to heavily restrict which parts of JSON-LD are allowed.
This creates a lose-lose situation: JSON-LD experts may be confused by the deviations from standard JSON-LD, while users unfamiliar with JSON-LD would still need to learn both JSON-LD concepts and the OME-NGFF-specific extensions.
The learning curve would be higher than with a purpose-built format, without corresponding benefits in return.

For these reasons, the RFC adopts a simpler custom format and a custom type system which, while somewhat reminiscent of JSON-LD on some aspects (the `extension_manifest` is partially akin to a JSON-LD `@context`), is distinct enough not to mislead users into believing that OME-Zarr JSON files are actually JSON-LD files.

### Custom types identified by prefixed keywords

An initial version of this RFC proposed that third parties could extend the specification by (1) registering an extension prefix into a registry maintained by the community and (2) prepend the registered prefix to any type name they wish to create.

This was abandoned in favour of (1) mandating that custom types be identified by an identifier that is derived from an _extension_ identifier, where the extension is itself declared in the _extension manifest_; and (2) recommending that extension identifiers (and therefore custom type identifiers) be URI-based. This better meets the [desired characteristics](#desired-characteristics-for-ome-zarrs-extensibility-mechanism) for OME-Zarr extensibility, as set forth in the June 2026 Heidelberg hackathon.

### Prefixed attribute keys

An initial version of this RFC proposed that third parties could append custom metadata to any node by adding prefixed keys to an `attributes` dictionary carried by every node, where the prefix has been previously registered into a registry maintained by the community (as for custom types, see previous section).

This was abandoned in favour of the [trait](#traits) concept.


<!--
As RFCs evolve, it is common that there are ideas that are abandoned. Rather
than simply deleting them from the document, you should try to organize them
into sections that make it clear they're abandoned while explaining why they
were abandoned.

When sharing your RFC with others or having someone look back on your RFC in
the future, it is common to walk the same path and fall into the same pitfalls
that we've since matured from. Abandoned ideas are a way to recognize that path
and explain the pitfalls and why they were abandoned.

-->

## Prior art and references

### Initial Github issue

Initial discussions started here: https://github.com/ome/ngff/issues/31

### Neuroglancer JSON

See https://neuroglancer-docs.web.app/json/api/index.html

<details>
<summary>JSON example</summary>
Shortened version from https://fafb-ffn1.storage.googleapis.com/landing.html
<pre>
{
  "dimensions": {
    "x": [
      4e-9,
      "m"
    ],
    "y": [
      4e-9,
      "m"
    ],
    "z": [
      4e-8,
      "m"
    ]
  },
  "position": [
    109421.8984375,
    41044.6796875,
    5417
  ],
  "crossSectionScale": 2.1875,
  "projectionOrientation": [
    -0.08939177542924881,
    -0.9848012924194336,
    -0.07470247149467468,
    0.12882165610790253
  ],
  "projectionScale": 27773.019357116023,
  "layers": [
    {
      "type": "image",
      "source": {
        "url": "precomputed://gs://neuroglancer-fafb-data/fafb_v14/fafb_v14_orig"
      },
      "blend": "default",
      "name": "fafb_v14",
      "visible": false
    },
    {
      "type": "image",
      "source": {
        "url": "precomputed://gs://neuroglancer-fafb-data/fafb_v14/fafb_v14_clahe"
      },
      "blend": "default",
      "name": "fafb_v14_clahe"
    },
    {
      "type": "segmentation",
      "source": {
        "url": "precomputed://gs://fafb-ffn1-20190805/segmentation"
      },
      "segments": [
        "710435991"
      ],
      "skeletonRendering": {
        "mode2d": "lines_and_points",
        "mode3d": "lines"
      },
      "name": "fafb-ffn1-20190805"
    },
    ...
  ],
  "showAxisLines": false,
  "showSlices": false,
  "layout": "xy-3d"
}
</pre>
</details>


### MoBIE collections

The MoBIE collection table allows users to specify a collection of images and segmentations (label mask images) and configure their rendering.

Each row in the table corresponds to one (single-channel) image or segmentation.

To open multi-channel images the image URI must be added several times and a `channel` column must be added to specify which channel to load.

One can specify an affine transformation for each image.

One can configure a grid layout to render multiple images side-by-side.

The same image can be opened multiple times, with different rendering settings or different transformations.

- [Example table](https://github.com/mobie/mobie-viewer-fiji/blob/main/src/test/resources/collections/blobs-grid-table.txt)
- [More example tables](https://github.com/mobie/mobie-viewer-fiji/blob/main/src/test/resources/collections)
- [More details](https://mobie.github.io/tutorials/mobie_collection_table.html)


### Webknossos datasource-properties.json
Webknossos uses a JSON format to define "datasets", which is a non-nestable collection of images with some metadata attached. It mirrors some of the OME-Zarr multiscale metadata for compatibility reasons.

<details>
<summary>JSON example</summary>
<pre>
{
  "id": {
    "name": "l4_sample3",
    "team": ""
  },
  "dataLayers": [
    {
      "name": "color",
      "category": "color",
      "boundingBox": {
        "topLeft": [
          3072,
          3072,
          512
        ],
        "width": 1024,
        "height": 1024,
        "depth": 1024
      },
      "elementClass": "uint8",
      "mags": [
        {
          "mag": [
            1,
            1,
            1
          ],
          "path": "./color/1",
          "axisOrder": {
            "x": 1,
            "y": 2,
            "z": 3,
            "c": 0
          },
          "channelIndex": 0
        },
        {
          "mag": [
            2,
            2,
            1
          ],
          "path": "./color/2-2-1",
          "axisOrder": {
            "x": 1,
            "y": 2,
            "z": 3,
            "c": 0
          },
          "channelIndex": 0
        }
      ],
      "numChannels": 1,
      "dataFormat": "zarr"
    },
    {
      "name": "segmentation",
      "boundingBox": {
        "topLeft": [
          3072,
          3072,
          512
        ],
        "width": 1024,
        "height": 1024,
        "depth": 1024
      },
      "elementClass": "uint32",
      "mags": [
        {
          "mag": [
            1,
            1,
            1
          ],
          "path": "./segmentation/1",
          "axisOrder": {
            "x": 1,
            "y": 2,
            "z": 3,
            "c": 0
          },
          "channelIndex": 0
        },
        {
          "mag": [
            2,
            2,
            1
          ],
          "path": "./segmentation/2-2-1",
          "axisOrder": {
            "x": 1,
            "y": 2,
            "z": 3,
            "c": 0
          },
          "channelIndex": 0
        }
      ],
      "largestSegmentId": 2504697,
      "numChannels": 1,
      "dataFormat": "zarr",
      "category": "segmentation"
    }
  ],
  "scale": {
    "factor": [
      11.24,
      11.24,
      28
    ],
    "unit": "nanometer"
  }
}
</pre>
</details>

### STAC
from the Geo Community https://stacindex.org/ https://stacspec.org/

### Separate declaration and reference keys

Several existing formats use different keys for declaring an object and for referring to it, as the [declarations](#declaration-interface) and [references](#reference-interface) in this RFC do:

- **[OME-XML](https://www.openmicroscopy.org/Schemas/)** declares objects with an `ID` attribute (e.g. `<Image ID="Image:0">`) and refers to them with dedicated `…Ref` elements (e.g. `<ImageRef ID="Image:0"/>`).
- **[FHIR](https://hl7.org/fhir/references.html)** declares resources with an `id` and refers to them with a `Reference` data type whose `reference` field names the type and ID of the target (e.g. `"reference": "Patient/123"`).
- **[STAC](https://github.com/radiantearth/stac-spec)** declares catalogs, collections and items with an `id` and links between them with link objects that carry an `href` and a `rel` describing the relationship.

### BIDS
https://direct.mit.edu/imag/article/doi/10.1162/imag_a_00103/119672/The-Past-Present-and-Future-of-the-Brain-Imaging

https://github.com/bids-standard/bids-specification-pdf-releases/blob/1.10.0/bids-spec.pdf

People worked on getting ome-zarr into BIDS
https://github.com/bids-standard/bids-examples/tree/master/micr_XPCTzarr

### OMERO.figure
OMERO.figure uses a JSON file to specify the layout and rendering settings for images, as well as some info on page(s) layout. Until now, these images are hosted and rendered by OMERO, but support for OME-Zarr images is underway at https://github.com/ome/omero-figure/pull/619. Here, the OMERO Image ID is replaced by an OME-Zarr URL and rendering happens in the browser. See [the demo](https://will-moore.github.io/omero-figure/?file=https://gist.githubusercontent.com/will-moore/fe0e260544b46af6e1e523b288fc85bc/raw/30547e61d4d8753ef0016f0a70435f1aafb43c2f/OMERO.figure_NGFF_demo.json).

The OMERO.figure json format is [described here](https://github.com/ome/omero-figure/blob/master/docs/figure_file_format.rst#json-format).

<details>
<summary>JSON example</summary>
Shortened version from https://gist.githubusercontent.com/will-moore/fe0e260544b46af6e1e523b288fc85bc/raw/30547e61d4d8753ef0016f0a70435f1aafb43c2f/OMERO.figure_NGFF_demo.json
<pre>
{
    "panels": [
        {
            "x": 32.776939633609516,
            "y": 299.60384546911166,
            "width": 157,
            "height": 160.5839416058394,
            "zoom": 100,
            "dx": 0,
            "dy": 0,
            "labels": [],
            "deltaT": [],
            "rotation": 0,
            "selected": false,
            "pixel_size_x_symbol": "µm",
            "pixel_size_x_unit": "MICROMETER",
            "rotation_symbol": "°",
            "max_export_dpi": 1000,
            "vertical_flip": false,
            "horizontal_flip": false,
            "imageId": "https://uk1s3.embassy.ebi.ac.uk/idr/zarr/v0.4/idr0062A/6001240.zarr",
            "name": "6001240.zarr",
            "sizeZ": 236,
            "theZ": 118,
            "sizeT": 1,
            "theT": 0,
            "channels": [
                {
                    "active": true,
                    "coefficient": 1,
                    "color": "0000FF",
                    "family": "linear",
                    "inverted": false,
                    "label": "LaminB1",
                    "window": {
                        "end": 1500,
                        "max": 65535,
                        "min": 0,
                        "start": 0
                    }
                },
                {
                    "active": true,
                    "coefficient": 1,
                    "color": "FFFF00",
                    "family": "linear",
                    "inverted": false,
                    "label": "Dapi",
                    "window": {
                        "end": 1500,
                        "max": 65535,
                        "min": 0,
                        "start": 0
                    }
                }
            ],
            "orig_width": 271,
            "orig_height": 275,
            "pixelsType": "uint16",
            "zarr": {
                // consolidate zarr metadata here
            }
        }
    ]
}
</pre>
</details>

### Rust traits

[Traits](#traits), as used in this RFC, were inspired by the [concept of the same name](https://doc.rust-lang.org/book/ch10-02-traits.html) in the Rust programming language.

### Extensions

The design of the [extensibility mechanism](#extensions) was influenced by:

- the [OME-Zarr Extensibility Manifesto](https://github.com/gouttegd/yamf-playground/blob/main/ome-zarr/EXTENSIBILITY_MANIFESTO.md), which is itself the result of a discussion that took place in an OME-Zarr hackathon held in Heidelberg in June 2026; the [“Rationale” section](#rationale-for-ome-zarr-extensions) in the present RFC is essentially a condensed version of that document;
- the [Proposal for a generic extensibility scheme](https://github.com/gouttegd/yamf-playground/blob/main/EXTENSIBILITY.md) – what this RFC calls a “trait” is essentially what that proposal called a “class extension”.

<!--

Is there any background material that might be helpful when reading this
proposal? For instance, do other operating systems address the same problem
this proposal addresses?

Discuss prior art, both the good and the bad, in relation to this proposal. A
few examples of what this can include are:

Does this feature exist in other formats and what experiences has their
community had?

Are there any published papers or great posts that discuss this? If you have
some relevant papers to refer to, this can serve as a more detailed theoretical
background.

This section is intended to encourage you as an author to think about the
lessons from other domains, and provide readers of your RFC with a fuller
picture. If there is no prior art, that is fine - your ideas are interesting to
us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does
not on its own motivate an RFC.

-->

## Future possibilities

The collections mechanism is a possibility for extending OME-Zarr in a backwards-compatible manner.

Future additions include:

- Additional node types, such as tables, meshes
- Additional path formats, such as S3 URLs
- A `source` trait on any node ("derived from"), and a node type for data that is not OME-Zarr (a path, possibly into an archive, and the data's own coordinate systems), so derived images can name what they came from across the format boundary (user stories 3, 9, 11)
- Traits stating the extent of a node per resolution level and whether absent chunks are unwritten or fill, so sparse and sub-region images render correctly (user story 11)
- A rule that a node's `id` is the identifier a provenance record (such as the `provenance:` Zarr convention) uses for it, so that process history and node relationships can be joined (user story 11)

Within the collections proposal, there are also opportunities for standardizing further metadata:

- Rendering state
- Expressing relationships between nodes in a collection

Future work:

- Defining formal mechanism for extensions
- Implementing the [URL pipeline specification](https://github.com/jbms/url-pipeline) proposed by Jeremy Maitin-Shephard

<!--
Think about what the natural extension and evolution of your proposal would be
and how it would affect the specification and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project in your proposal. Also consider how this all fits
into the roadmap for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC
you are writing but otherwise related. If you have tried and cannot think of
any future possibilities, you may simply state that you cannot think of
anything.

Note that having something written down in the future-possibilities section is
not a reason to accept the current or a future RFC; such notes should be in the
section on motivation or rationale in this or subsequent RFCs. The section
merely provides additional information.

-->

## Performance

**Multiple requests to collect metadata.** Assembling a collection from path-referenced nodes requires one metadata request per node (fetching the `zarr.json` of each referenced Zarr group or the standalone JSON file). For large collections or high-latency storage backends (e.g. S3, HTTP), this can result in significant startup latency. Implementations MAY mitigate this by fetching node metadata in parallel and SHOULD consider caching resolved node metadata.
Writers MAY additionally use Zarr consolidated metadata to bundle the `zarr.json` documents of referenced nodes into the root group, so that readers can load the whole tree in one request (see [Consolidated metadata](#consolidated-metadata-non-normative)).
The individual `zarr.json` files remain the source of truth.

<!--
What impact will this proposal have on performance? What benchmarks should we
create to evaluate the proposal? To evaluate the implementation? Which of those
benchmarks should we monitor on an ongoing basis?

Do you expect any (speed / memory)? How will you confirm?

There should be microbenchmarks. Are there?

There should be end-to-end tests and benchmarks. If there are not (since this
is still a design), how will you track that these will be created?

-->

## Compatibility

### Backwards compatible

The v1.0 goal is to replace the existing labels, HCS and
bioformats2raw.layout structures with the new extensibility framework. Expressing these data using RFC-8 extensibility framework will be discussed in topic-specific sub-RFCs. See the "Features deferred to subsequent sub-RFC” section for details.

### Forward compatible
Existing implementations need to be updated to be able to understand the new collection objects.

<!--
How does this proposal affect backwards and forwards compatibility?

Does it restrict existing assumptions or remove existing restrictions?

How are implementations expected to handle these changes?

-->

## Testing

As part of the changes to the OME-Zarr specification, JSON schema files will be provided that can be used to validate collections metadata.
Additional link checkers can be used to verify the existence and validity of nodes.

<!--
How will you test your feature? A typical testing strategy involves unit,
integration, and end-to-end tests. Are our existing test frameworks and
infrastructure sufficient to support these tests or does this proposal require
additional investment in those areas?

If your proposal defines a contract implemented by other people, how will those
people test that they have implemented the contract correctly? Consider, for
example, creating a conformance test suite for this purpose.

-->

## Tutorials and Examples

<!--

It is strongly recommended to provide as many examples as possible of what both users and developers can expect if the RFC were to be accepted. Sample data should be shared publicly. If longer-term is not available, contact the **Editors** for assistance.

-->

## Additional considerations

<!--
Most RFCs will not need to consider all the following issues. They are included here as a checklist
-->

### Security

This proposal allows collection metadata to reference other metadata documents through `Path` objects.
The core of this RFC only defines the `relative` path format, which introduces the security considerations below that implementations MUST address.

**Path traversal.** Relative paths (e.g. `../../sensitive/data`) may be used to reference files outside the intended storage scope.
Implementations SHOULD validate that resolved paths remain within an expected root directory or storage namespace, and MUST NOT follow paths that escape a defined sandbox without explicit user consent.

**Resource exhaustion.** Cyclic or very deeply nested paths may cause implementations to issue an unbounded number of requests.
Implementations MUST detect cycles as described in [Path resolution](#path-resolution) and MAY limit the depth to which they resolve nested paths.

<!--
What impact will this proposal have on security? Does the proposal require a
security review?

A good starting point is to think about how the system might encounter
untrusted inputs and how those inputs might be used to manipulate the system.
From there, consider how known classes of vulnerabilities might apply to the
system and what tools and techniques can be applied to avoid those
vulnerabilities.
-->

<!--
### Privacy

What impact will this proposal have on privacy? Does the proposal require a
privacy review?

A good starting point is to think about how user data might be collected,
stored, or processed by your system. From there, consider the lifecycle of such
data and any data protection techniques that may be employed.
-->


<!--

### UI/UX

If there are user- or frontend-impacting changes by this RFC, it is important
to have a "UI/UX" section. User-impacting changes might include changes in how
images will be rendered. Frontend-impacting changes might include the need to
perform additional preprocessing of inputs before displaying to users.

This section is effectively the "implementation" section for the user
experience. The goal is to explain the changes necessary, any impacts to
backwards compatibility, any impacts to normal workflow, etc.

As a reviewer, this section should be checked to see if the proposed changes
feel like the rest of the ecosystem. Further, if the breaking changes are
intolerable or there is a way to make a change while preserving compatibility,
that should be explored.

-->
