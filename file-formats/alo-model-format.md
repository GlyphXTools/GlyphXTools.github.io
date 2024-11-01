# Alamo Object Model File Format

## Introduction
Alamo Object Files (extension: .ALO) in Petroglyph's games describe a 3D model. It is a collection of bones, meshes with effects, proxies and dazzles. Alamo Object Files are an instance of [Chunked Files](chunk-format.md).

Note that .ALO files can also describe a [particle](alo-particle-format.md), but here we describe the format when they contain a 3D model.

Conceptually, an Alamo model consists of:
- A "skeleton": an acyclic tree of bones. Each bone defines a transformation relative to its parent. Bones can be modified by [animations](ala-animation-format.md). Objects in the model are 'attached' to a bone, meaning they are transformed with the bone's transformation.
- Meshes: named collections of vertices and indices rendered as triangles, separated into different sub-meshes with different materials.
- Lights: named light descriptions for e.g. omnidirectional or spot lights.
- Proxies: a reference to an external model or particle to be placed in the model.
- Dazzles: a blinking camera-aligned 2D sprite with customizable color and timing.

## Format
Each ALO model file contains several top-level chunks:
- one chunk describing the skeleton.
- any number of object chunks (meshes/lights).
- one chunk defining a list of proxies, dazzles and object connections.

### Skeleton
A skeleton is a top-level chunk (type `0x200`) and contains:
- one 'skeleton header' chunk (type `0x201`):
  ```
  dword     nBones
  byte[124] zero
  ```
  The header contains a 32-bit integer with the number of bones in the skeleton. The chunk data is zero-padded to 128 bytes.

- zero or more 'bone' chunks (type `0x202`). As many as indicated by the count in the header.
  Each bone chunk contains:
  - one 'bone name' chunk (type `0x203`). This holds a zero-terminated ASCII string with the bone's name.
  - one 'bone data' chunk (type `0x205` or type `0x206`).
    ```
    int       parent
    dword     visible
    dword     billboard   # missing in chunks with type 0x205
    float4[3] matrix
    ```
    This chunk holds the bone information; the parent bone index (which is always less than this bone's index), whether or not the meshes and/or proxies attached to this bone (and its children) are visible, the [billboard mode](../articles/understanding-billboarding.md), and the transformation matrix.

    Note that the matrix, stored row-wise, contains 3 rows of 4 columns each. After reading these rows, the 4th row should be (0,0,0,1) to create a transposed 4x4 matrix that defines this bone's local transformation (i.e., relative to its parent).

    For chunks with type `0x205`, the 'billboard' field is missing. When reading those chunks, assume the billboard field is zero.

### Meshes
A mesh is a top-level chunk (type `0x400`) and defines a named collection of sub-meshes that create a cohesive whole.
Each mesh chunk contains:
- one 'mesh name' chunk (type `0x401`). This holds a zero-terminated ASCII string with the mesh's name.
- one 'mesh info' chunk (type `0x402`):
  ```
  dword     nMaterials
  float3[2] boundingBox
  dword     unused
  dword     isHidden
  dword     isCollisionEnabled
  byte[88]  zero
  ```
  This chunk contains basic information about the mesh: the number of materials (sub-meshes; 0x10100 and 0x10000 chunk pairs) in this mesh, the bounding box for this mesh, whether this mesh is initially hidden and whether this mesh can be collided with (i.e., whether the mesh's sub-meshes have collision trees (`0x1200` chunks)). The bounding box is stored as 6 floating point values: two X,Y,Z triples for the minimum and maximum points of the box, respectively.
  The chunk data is zero-padded to 128 bytes.

- one or more sub-mesh material chunk pairs (types `0x10100` and `0x10000`). The amount is indicated by 'nMaterials' in the mesh info.

#### Mesh Materials
The data in a mesh is stored as a collection of "sub-meshes", one per material.
For each, two chunks are stored: the 'sub-mesh material' chunk (type `0x10100`) and the 'sub-mesh data' chunk (type `0x10000`).

##### Sub-mesh material

Sub-mesh material chunks (type `0x10100`) are chunks that identify the material's shader file and its parameters to be used when rendering this mesh and its parameters.

Shader parameters have a name, type and data and should be used with the [GetParameterByName](http://msdn2.microsoft.com/en-us/library/bb205696.aspx) and [SetInt](http://msdn2.microsoft.com/en-us/library/bb205718.aspx), [SetFloat](http://msdn2.microsoft.com/en-us/library/bb205716.aspx), [SetFloatArray](http://msdn2.microsoft.com/en-us/library/bb205717.aspx), [SetTexture](http://msdn2.microsoft.com/en-us/library/bb205727.aspx), etc. functions.

These chunks contain:
- one 'material name' chunk (type `0x10101`). This chunk contains the material name as an ASCIIZ string. Resolving this name is engine-dependent, but usually involves loading a graphics shader file of the same name, possibly with different extensions (e.g. '.fx', '.fxo').
- zero or more shader parameters chunks:
  - INT (chunk type `0x10102`):
    ```
    01: string name
    02: dword  value
    ```
    This chunk contains two mini-chunks of types `1` and `2`: the parameter's name and value, respectively.
  
  - FLOAT (chunk type `0x10103`):
    ```
    01: string name
    02: float  value
    ```
    This chunk contains two mini-chunks of types `1` and `2`: the parameter's name and value, respectively.

  - FLOAT3 (chunk type `0x10104`):
    ```
    01: string name
    02: float3 value
    ```
    This chunk contains two mini-chunks of types `1` and `2`: the parameter's name and value, respectively.

  - TEXTURE (chunk type `0x10105`):
    ```
    01: string name
    02: string value
    ```
    This chunk contains two mini-chunks of types `1` and `2`: the parameter's name and value, respectively. The value indicates the name of a texture. Resolving this name is engine-dependent, but usually involves loading a texture of the same name, possibly with different extensions (e.g. '.tga', '.dds', etc).

  - FLOAT4 (chunk type `0x10106`):
    ```
    01: string name
    02: float4 value
    ```
    This chunk contains two mini-chunks of types `1` and `2`: the parameter's name and value, respectively.

##### Sub-mesh data

Sub-mesh data chunks (type `0x10000`) contain the core sub-mesh data: vertices, indices, animation mappings and collision tree. These chunks contain:
- one 'sub-mesh data header' chunk (type `0x10001`):
  ```
  dword     nVertices
  dword     nPrimitives
  byte[120] zero
  ```
  The chunk simply holds a header with the number of vertices and triangles in the mesh. The chunk's data is zero-padded to 128 bytes.

- one 'sub-mesh vertex format' chunk (type `0x10002`). This holds a zero-terminated ASCII string with the vertex format. The vertices in the vertex buffer chunk are always full vertices, but this string indicates which subset is used by the shader. The engine can thus only populate a small vertex buffer in memory. An example vertex format is "alD3dVertNU2U3U3C", which indicates a vertex with position, normal vector ("N"), UV coordinates ("U2"), tangent coordinates ("U3"), binormal coordinates ("U3") and RGBA color ("C").

- one 'index buffer' chunk (type `0x10004`):
  ```
  word[3 * header.nPrimitives] indices
  ```
  This chunk contains the index buffer that is used to render the primitives of the sub-mesh. The number of indices in the array is specified in the `0x10001` chunk. The index buffer is an array of 16-bit indices into the vertex buffer located in the same parent chunk.

- one 'vertex buffer' chunk (type `0x10005` or type `0x10007`):
  ```
  struct Vertex {
    float3    position
    float3    normal
    float2[4] texCoords
    float3    tangent
    float3    binormal
    float4    color
    float4    unused        # missing in chunks with type 0x10005
    dword[4]  boneIndices
    float[4]  boneWeights
  };
  
  Vertex[header.nVertices] vertices
  ```
  This chunk holds an array of vertices, where each vertex has the structure as described above. The number of vertices in this array is specified in the `0x10001` chunk. Note however, that although there is space for 4 bone-indices, only the first is every used. Therefore, boneWeights is always [1, 0, 0, 0]. These bone indices are indices into the animation mapping array (chunk `0x10006`).

- zero or one 'animation mapping' chunk (type `0x10006`):
  ```
  dword[1..24] bone
  ```
  [Alamo animations](ala-animation-format.md) modify the transformations of the bones in the skeleton. Some mesh materials are "rigid": all vertices in the sub-mesh are attached to a single bone (e.g. a gun being held in a hand).  Other materials are "skinned": different vertices in the sub-mesh are attached to different bones (e.g. a single 'body' sub-mesh where the vertices of an arm are attached to the arm bones).
  
  For "skinned" materials/shaders, each vertex defines the bone it is connected to. This value is an index into this array. All vertices in a single sub-mesh can be affected by no more than 24 bones of the skeleton. This chunk defines which bones of the skeleton affect this sub-mesh. It contains between 1 and 24 32-bit integers that are indices into the bones in the skeleton (`0x200` chunk).

- one 'collision tree' chunk (type `0x1200`). This chunk, which is only present if the mesh information (chunk type `0x402`) chunk's `isCollisionEnabled` field is non-zero, holds a spatial tree that can be traversed in logarithmic time with integer comparisons to do collision detection. This chunks contains:
  
  - one 'collision tree header' chunk (type `0x1201`):
    ```
    00: float3 collisionBoxMin
    01: float3 collisionBoxMax
    02: dword  nNodes
    03: dword  nPrimitives
    ```
    This chunk contains four mini-chunks which describe the reference box for the collision detection (`00` and `01` mini-chunks) and define the number of records in the `0x1202` and `0x1203` chunks (`02` and `03` mini-chunks, respectively).

  - one 'collision tree nodes' chunk (type `0x1202`):
    ```
    struct Node {
      byte3 min
      byte3 max
      word  nPrimitives
      word  link
    };

    Node[header.nNodes] nodes;
    ```
    This chunk contains a list of collision tree nodes that make up the tree. Each node is associated with a sub-region of its container box. The coordinates of this sub-box are stored as 8-bit values. These values are normalized to the volume of the node's parent box. In other words, the coordinate span (0,0,0) - (255,255,255) indicates the same box as the parent node.

    If `nPrimitives > 0` for a node, the `link` field is an index into the triangle mapping array defined in the `0x1203` chunk and the `nPrimitives` value indicates how many subsequent triangles are contained in this leaf.

    If `nPrimitives == 0` for a node, the `link` value is an index into the node array in this chunk, identifying the two children of this node. In other words, the children of node `x` are defined as the nodes with indices `nodes[x].link` and `nodes[x].link + 1`.

  - one 'collision tree triangle mapping' chunk (type `0x1203`):
    ```
    word[nPrimitives] triangles;
    ```
    This chunk contains an array of 16-bit integers that identify triangles in the model (via the sub-mesh's index buffer). These triangle references are used from the `0x1202` chunk to construct a collision tree.

### Lights
A light is a top-level chunk (type `0x1300`) and contains:
- one 'light name' chunk (type `0x1301`). This holds a zero-terminated ASCII string with the light's name.
- one 'light info' chunk (type `0x1302`):
  ```
  dword  type
  float3 color
  float  intensity
  float  farAttenuationEnd
  float  farAttenuationStart
  float  hotspotSize
  float  falloffSize
  ```
  This chunk stores the light information. The type can be `0` (Omni light), `1` (Directional light) or `2` (Spotlight). The color is stored as an RGB triplet with each element in the range 0.0 - 1.0. Furthermore, the `hotspotSize` and `falloffSize` are both in radians.
  
### Connections
The connections chunk is a top-level chunk (type `0x600`) and defines which bones the meshes, lights and proxies are connected to. This chunk contains:
- one 'connections header' chunk (type `0x601`):
  ```
  01: dword nObjects
  04: dword nProxies
  09: dword nDazzles     # optional
  ```
  This chunk contains two or three mini-chunks, each with a single 32-bit integer, that indicate the number of `0x602` and `0x603` chunks, respectively.

- zero or more 'object connection' chunks (type `0x602`):
  ```
  02: dword object
  03: dword bone
  ```
  This chunk contains two minichunks with the index of an object (mesh or light) and the index of a bone to indicate that the specified mesh or light is connected to the position of the specified bone. Mesh/light indices are determined from the order in which the `0x400` and `0x600` chunks appear in the file.

- zero or more 'proxy connection' chunks (type `0x603`):
  ```
  05: string proxy
  06: dword  bone
  07: dword  isHidden
  08: dword  altDecreaseStayHidden
  ```
  This chunk contains four minichunks which identify the name of another ALO object (which can be either a model or a particle effect) to be loaded and attached to the specified bone.
  The `isHidden` and `altDecreaseStayHidden` values indicate if this proxy is initially hidden and if the "Alt Decrease Stay Hidden" property is set for this proxy (see section below).

- zero or more 'dazzle' chunks (type `0x604`):
  ```
  00: float3 color
  01: float3 position
  02: float  radius
  03: dword  texX
  04: dword  texY
  06: dword  texSize
  05: string texture
  07: float  frequency
  08: float  phase
  09: dword  nightOnly
  0a: dword  bone
  0b: string name
  0c: dword  isHidden
  0d: float  bias       # optional
  ```
  This chunk contains the information for a single "dazzle" object. A dazzle is a camera-facing square sprite, generally rendered with an emissive material, whose visibility fluctuates over time according to a sine wave (imitating a blinking effect)
  The fields above are used as follows:
  - The `name` property identifies the dazzle. This has little practical use, except for one thing: if the name starts with "FC_", the dazzle should be _colorized_ (see section "Colorization", below).
  - The `bone`, `position` and `radius` fields define the bone the dazzle is connected to, its relative offset to the bone and its world-space radius (i.e. half the width/height of the sprite).
  - The `color` field defines the RGB color of the dazzle. This may be amended by colorization (see section "Colorization", below).
  - The `texture` field names the texture to use for the sprite, while the `texSize`, `texX` and `texY` fields identify the part of the texture. The texture is assumed to be square, consisting of `texSize * texSize` equally-sized elements.
    `texX` and `texY` identify the element in this 'grid' of texture elements.
  - The `frequency`, `phase` and `bias` fields determine the periodic visibility of the dazzle: `frequency` determines how often it 'blinks' per second. `phase` determine the phase offset (from 0 to 1). `bias` (if present) indicates the point in the blink phase where the dazzle is hidden. E.g. a bias of 0.4 means the dazzle is only shown for the first 40% of each 'blink'.
    Taken together, the dazzle's visibility at game time _t_ (in seconds) is: _sin(a&nbsp;π)_ if _a&nbsp;&le;&nbsp;bias_ else _0_, with _a&nbsp;=&nbsp;(t&nbsp;⋅&nbsp;frequency&nbsp;+&nbsp;phase)&nbsp;mod&nbsp;1_.

    If frequency is 0, the dazzle is always shown at 100% visibility.
  - The `nightOnly` field identifies if the dazzle is only shown during in-game night time.
  - The `isHidden` property indicates if this dazzle is initially hidden.

## Mesh LODs and Alternatives
An Alamo object can contain the same mesh in different levels of details (LOD) or level-of-damage "alternatives" ("ALTs").

These are stored as separate meshes (`0x400` chunks) with the same base name, but suffixed with "`_LOD<n>`" and/or "`_ALT<n>`". For instance, "Unit_LOD0" (low-poly version), "Unit_LOD1" (medium-poly version) and "Unit_LOD2" (high-poly version). Or "Building_LOD2_ALT0" (high-poly, healthy), "Building_LOD2_ALT1" (high-poly, damaged).

Meshes with higher LOD numbers are more detailed. Meshes with higher ALTs are more damaged. There can be any number of LODs and ALTs for a base mesh name.
Care must be taken (for relevant applications) to only render _one_ of the multiple meshes with the same base name, based on the current LOD and object health.

## Proxy alternatives
Like meshes, proxy objects can also be defined for different ALTs. For instance, two proxies, named "p_explosion01_ALT1" and "p_explosion02_ALT2" can be defined to show "p_explosion01" when the object becomes lightly damaged and "p_explosion02" when the object becomes heavily damaged.

These proxy objects are shown when the object's ALT level (based on its health) becomes the ALT level specified in the name. For effects like explosions, this should be avoided when the object "heals" (i.e. going from ALT2 to ALT1). For these cases, proxy connections have their "Alt Decrease Stay Hidden" property set to true: in this case, those proxy objects are not shown if the ALT level _decreases_ instead of _increases_.

## Colorization
Objects can be colorized. This means that part of the object's texture are "colored" with the mesh's object's owning faction's color. This is how two identical objects owned by different factions can be told apart.

For meshes, depending on the game, there's several ways to do this:
* There's a hard-coded material shader parameter name such as "Colorization" that's replaced by the rendering engine with the mesh's object's owning faction's color.
* If the mesh name starts with "FC_", it should be colorized. The rendering engine then replaces the "Color" material shader parameter with the mesh's object's owning faction's color.
In both cases, the shader then applies that colorization to the material (typically by modulating the texture color with this colorization parameter). 

Dazzles can also be colorized. This should be done if their name starts with "FC_". Colorization of dazzles happens in [HSV](https://en.wikipedia.org/wiki/HSL_and_HSV) space: the dazzle's color is converted to HSV and its hue is replaced with the colorization's hue. The result is then converted back to RGB for rendering.

## Game support notes
Star Wars: Empire at War and its expansion, Forces of Corruption, do not understand dazzles.

