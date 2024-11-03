# Alamo Object Particle File Format

## Introduction
Alamo Particle files (extension: .ALO) in Petroglyph's games describe a _particle system_. A particle system is a collection of particle emitters, that each emit a certain type of particle at a certain interval. Particles are camera-facing 2D sprites that can independently move and change in size and color after being emitted. Alamo Particle Files are an instance of [Chunked Files](chunk-format.md).

Note that .ALO files can also describe a [3D model](alo-model-format.md), but here we describe the format when they contain a particle.

## Format
An Alamo Particle file contains a single top-level chunk of type `0x900`, which contains:
- one 'particle system name' chunk (type `0x0`). This holds a zero-terminated ASCII string with the particle system's name. This name **must** match the filename without extension.

- one 'particle system ID' chunk (type `0x1`):
  ```
  uint32 id
  ```
  This chunk contains the particle system's ID. This is a value used internally in the editor and can be disregarded.

- one 'particle system persistence' chunk (type `0x2`):
  ```
  byte persist
  ```
  If the 'persist' field contains a non-zero value, the particles emitted by this particle system's emitters continue to exist to finish simulating even after the particle system has been removed. If the value is zero, all emitted particles are immediately removed along with the particle system.

- one 'particle emitters' chunk (type `0x800`). This chunk contains multiple particle emitters (see below).

### Particle emitter
A 'particle emitter' chunk (type `0x700`) describes a single particle emitter. An emitter spawns particles at a certain interval with certain properties such as size, offset, velocity and color. These properties can be randomized for each emitted particle within customizable ranges.
This chunk contains:
- one 'particle emitter properties' chunk (type `0x2`):
  ```
  04: uint32   blendMode
  05: uint32   primitiveType
  06: uint32   unused1
  07: byte     inBursts
  08: byte     linkToSystem
  09: float[3] inwardSpeed
  10: float[3] acceleration
  11: float    outwardAcceleration
  12: float    gravity
  15: float    lifetime
  16: uint32   numTextureElements
  17: float    unused2
  18: float    randomizedScale
  19: float    randomizedLifetime
  20: uint32   index
  21: byte     unused3
  23: float    randomizedRotation
  35: byte     rotationDirection
  36: float    initialDelay
  37: float    burstDelay
  38: uint32   numParticlesPerBurst
  39: uint32   numBursts
  40: float    emitterSpeedMult
  42: uint32   numParticlesPerSecond
  43: byte     unused4
  44: float[4] randomizedColor
  45: byte     randomizedIsGrayscale
  46: byte     isNotBillboarded
  47: uint32   groundInteraction
  48: float    bounciness
  49: byte     affectedByWind
  50: float    freezeTime
  51: float    skipTime
  52: uint32   emitMode
  53: byte     objectSpaceAcceleration
  59: byte     isHeatParticle
  60: float    emitOffset
  61: byte     isWeatherParticle
  62: float    weatherCubeSize
  63: float    unused5
  64: float    unused6
  65: byte     hasTail
  66: float    tailSize
  67: byte     useEmitterSpeedMult
  68: byte     windDisturbance
  70: byte     noDepthTest
  71: float    weatherCubeDistance
  72: byte     fixedRotation
  ```
  This chunk describes the main properties of the emitter:
  - `index` contains the index of the emitter in the list of emitters.
  - `inBursts` indicates whether particles are emitted in bursts (all at once), or as a continuous stream of single particles:
    - If `true`, the emitter emits a burst of `numParticlesPerBurst` particles every `burstDelay` seconds until `numBursts` bursts have been emitted (or, if `numBursts` is zero, until the emitter dies).
    - If `false`, the emitter continuously emits particles at a rate of `numParticlesPerSecond` per second until the emitter dies.
  - `initialDelay` specifies the time, in seconds since emitter creation, until the first particle is emitted.
  - `freezeTime` specifies the time, in seconds since emitter creation, after which particle simulation should stop simulating, and leave all particles as they are until the emitter dies.
  - `skipTime` specifies the time, in seconds since emitter creation, that the emitter should skip ahead at the start. It effectively simulates this much time, meaning that the particle simulation has instantly advanced to this point when the emitter is created. Note that `initialDelay` and `freezeTime` are included in the skip time.
  - `maxLifetime` and `randomizedLifetime` describe the lifetime, in seconds, of emitted particles. When emitted, a particle's lifetime is a random value between `maxLifetime * (1 - randomizedLifetime)` and `maxLifetime`.
  - If `linkToSystem` is true, the emitter's coordinate system is transformed to its parent's coordinate system. This effectively makes the emitter act relative to its parent (which can be a particle or mesh).
  - `emitMode` specifies how the particle is emitted from the mesh it is attached to (if any): 0=none (particle ignores the attached mesh and emits normally), 1=random_vertex (particle is emitted from a random vertex of the mesh), 2=random_point (particle is emitted from a random point on the mesh surface), 3=all_vertices (particle is emitted from every vertex at the mesh at the same time, i.e. the emitter is duplicated for every vertex). For 'random_vertex' and 'random_point', `emitOffset` describes the distance (in world units) along the normal at the emission point where the particles should be emitted.
  - If `useEmitterSpeedMult` is true, emitted particles inherit the emitter's parent's velocity, multiplied by `emitterSpeedMult`.
  - `inwardSpeed`, if non-zero, determines the speed of emitted particles (in world units/second) towards the position of the emitter.
  - `acceleration` and `objectSpaceAcceleration` determine the base acceleration vector of emitted particles. If `objectSpaceAcceleration` is true, the `acceleration` vector is relative to the emitter's parent's transformation. Otherwise, the `acceleration` vector is in world space.
  - `outwardAcceleration` determines the additional acceleration of emitted particles (in world units/second²) away from the position of the emitter.
  - `gravity` determines the additional **downwards** (negative Z) acceleration of emitted particles (in world units/second²). This is always in world space.
  - If `affectedByWind` is true, the environment's wind velocity is added to the particle velocity. If `windDisturbance` is also true, then the velocity of localized wind disturbances in the scene are also added to the particle's velocity.
  - `groundInteraction` specifies how the particle behaves when it hits the ground: 0=nothing (ground is ignored), 1=die (particle dies), 2=bounce (particle bounces back up, multiplying speed by `bounciness`), 3=stop (particle stops moving entirely)
  - `randomizedRotation` specifies the random variance in particle rotation speed. When rendering, the particle's rotation speed (in radians/second) is multiplied with a random number between `1 - randomizedRotation` and `1 + randomizedRotation`.
  - If `rotationDirection` is true, every particle's rotation direction is randomly chosen between clockwise or counter-clockwise on creation. Otherwise, it's always clockwise.
  - If `fixedRotation` is true, every particle's rotation is fixed upon creation.
  - `blendMode` indicates the blend mode during rendering: 0=opaque, 1=additive, 2=alpha, 3=modulate, 4=depthsprite_additive, 5=depthsprite_alpha, 6=depthsprite_modulate, 7=diffuse_alpha, 8=stencil_darken, 9=stencil_darken_blur, 10=heat, 11=particle_bump_alpha, 12=decal_bump_alpha, 13=alpha_scanlines. These blend modes typically correlate to specific shaders. However, if `isHeatParticle` the blend mode is overriden with 'alpha'.
  - `primitiveType` indicates whether to render the particles using triangles (`0`) or quads (`1`).
  - `numTextureElements` specifies the number of 'elements' in the particle textures (see chunk `0x3`). The particle is rendered with a subset of the textures by selecting an 'element' from the texture by index.
  - `randomizedScale` specifies the random variance in particle size. When rendering, the particle's size is multiplied with a random number between `1 - randomizedScale` and `1 + randomizedScale`. This number is chosen when the particle is created.
  - `randomizedColor` is a vector of RGBA percentages. Upon creation, a random sRGBA color addition in the range _[0,&nbsp;100&nbsp;⋅&nbsp;`randomizedColor.[rgba]`]_ is added to the particle's color (capped at 255). If `randomizedIsGrayscale` is true, the random color addition's RGB components are all the same.
  - If `hasTail` is true, the particle is stretched backwards along its velocity vector (in screen space). `tailSize` is a scaling factor for the length of the particle's "tail".
  - If `noDepthTest` is true, the particles are always rendered, even if they would lie behind obstructing geometry. i.e., their depth-test is disabled.
  - If `isNotBillboarded` is true, the particles are rendered 'flat' on the XY plane instead of camera-facing ("billboarded").
   - `isWeatherParticle` indicates the particle is a _weather particle_. Weather particles have special rules and override multiple other properties. They are emitted at random positions in a cube where each side has length `weatherCubeDistance`. This cube is locked to the camera and positioned `weatherCubeDistance` world units in front of the camera (along the view axis). Note that once a particle has been emitted, it moves independently from the camera.

- one 'particle emitter color texture name' chunk (type `0x3`). This holds a zero-terminated ASCII string with the emitter's particles' color texture's name.

- one 'particle emitter name' chunk (type `0x16`). This holds a zero-terminated ASCII string with the emitter's name. This has no purpose other than a human-readable description of the emitter for e.g. an editor application.

- one 'particle emitter groups' chunk (type `0x29`). This chunk contains property groups for the emitter particle's **initial velocity**, **lifetime** and **position** (in that order). These groups describe a volume from which these three properties are randomly chosen.

  Note that the **lifetime** property group is redundant and must be filled with the lifetime info from the 'particle emitter properties' chunk as follows: `type = 1; minX = maxX = minZ = maxZ = 0; minY = lifetime * (1 - randomizedLifetime); maxY = lifetime`.

  This chunks contains:
  - three 'particle emitter group' chunks (type `0x1100`). This is just a container for:
    - one 'particle emitter group data' chunk (type `0x1101`):
      ```
      uint32 type
      float  minX, minY, minZ
      float  maxX, maxY, maxZ
      float  sideLength
      float  sphereRadius
      uint32 sphereSurface
      float  cylinderRadius
      uint32 cylinderSurface
      float  cylinderHeight
      float  valX, valY, valZ
      ```
      This chunk holds the data of the property group. There are several types of groups that can be stored, that behave differently and use different fields:
      - The **exact** group (`type = 0`): the property always has the same 3D value: (`valX`, `valY`, `valZ`)
      - The **box** group (`type = 1`): the property's 3D value is a randomly chosen point in the volume of a box with extents (`minX`, `minY`, `minZ`) and (`maxX`, `maxY`, `maxZ`).
      - The **cube** group (`type = 2`): the property's 3D value is a randomly chosen point in the volume of a cube of size `sideLength`, with the center of the box at (0,0,0).
      - The **sphere** group (`type = 3`): the property's 3D value is a randomly chosen point in the volume of a sphere centered at (0,0,0) with radius `spehereRadius`. If `sphereSurface` is true, the random point must be chosen on the sphere's surface.
      - The **cylinder** group (`type = 4`): the property's 3D value is a randomly chosen point in the volume of a cylinder with radius `cylinderRadius` and height `cylinderHeight` with its **bottom** circle centered at (0,0,0). Note that the Z range is thus [0, `cylinderHeight`]. If `cylinderSurface` is true, the random point must be chosen on the cylinder's (curved) surface. 

- one 'particle emitter tracks' chunk (type `0x1`). This chunks contains seven _tracks_: a collection of _(time, value)_ pairs describing the key points of a 2D graph. This graph provides the value for the track's associated property for a given time _t_ (as a fraction of the particles lifetime). This chunks contains the following two chunks, in order, repeated for each of seven properties: **red channel**, **green channel**, **blue channel**, **alpha channel**, **size**, **texture index**, and **rotation speed**:
  - 'particle track properties' (type `0x0`):
    ```
    02: float/byte firstKey
    03: float/byte lastKey
    04: uint32     interpolation
    ```
    This chunk contains three mini-chunks with the basic track information; the value for _t = 0_, the value for _t = 1_ and the interpolation method:
    - Linear (`interpolation = 0`): y = y1 + (y2 - y1) * (x - x1) / (x2 - x1)
    - Cosine (`interpolation = 1`): y = y1 + (y2 - y1) * (1 - cos(π ⋅ (x - x1) / (x2 - x1))) / 2
    - Step (`interpolation = 2`): y = (x &ge; x2) ? y2 : y1
    
    Note that the `02` and `03` mini-chunks contain an 8-bit unsigned integer (0 - 255) for the Red, Green, Blue and Alpha channel tracks and a float for the other three tracks.

  - 'particle track data' (type `0x1`):
    ```
    05: uint32/float value, float time
    05: uint32/float value, float time
    05: uint32/float value, float time
    ...
    ```
    This chunk contains a `05` mini-chunk for each key point in the graph. It can also be empty if the graph has no keys beyond the first and last. The value for each mini-chunk is a 64-bit block of data, where the first 32 bits are the value and the second 32 bits are the time. For the Red, Green, Blue and Alpha channel tracks, the value is a `uint32`. For the other three tracks, it's a `float`.

- one 'particle emitter connections' chunk (type `0x36`):
  ```
  37: uint32 death
  39: uint32 birth
  ```
  This chunk contains two mini-chunks each with an index of another emitter in this file. These emitters should be spawned on the death and birth of every particle from this emitter, respectively. The `birth` emitter should also be attached to the particle so that it moves along with the particle. For both cases, index -1 means 'no emitter'.

- one 'particle emitter secondary texture name' chunk (type `0x45`). This holds a zero-terminated ASCII string with the emitter's particles' secondary texture's name. Like the color texture, this texture is split into elements and the index is selected with the the value of the 'texture index' track.

  Depending on the shader, this texture is unused, a bump texture or a depth texture.
