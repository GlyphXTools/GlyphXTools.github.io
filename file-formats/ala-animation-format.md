# Alamo Animation File Format

Alamo Animation Files (extension: .ALA) in Petroglyph's games describe the transforms and changes in visibility of the bones in a [3D Alamo Model](alo-model-format.md).

Animation files define scale, translation, rotation, visibility and interpolation data for each bone, for a certain number of frames. Alamo Animation Files are an instance of [Chunked Files](chunk-format.md).

There are two known versions of this file format, both described below.

## Format #1

Each Alamo Animation file starts with a single top-level chunk, the animation file chunk (type `0x1000`). This chunk contains:
- one 'animation file header' chunk (type `0x1001`):
  ```
  01: dword numFrames
  02: float fps
  03: dword numBones
  ```
  This chunk contains three mini-chunks with the number of frames in the animation, the animation speed in frames per second and the number of bones for which animation data is stored (i.e., the number of `0x1002` chunks in the file).

- multiple 'bone animation data' chunks (type `0x1002`). Each chunk contains all animation information for a single bone and contains:
  - one 'bone animation data header' chunk (type `0x1003`):
    ```
    04: string name
    05: dword  index
    06: float3 translationOffset
    07: float3 translationScale
    08: float3 scaleOffset
    09: float3 scaleScale
    10: dword  unknown
    ```
    This chunk holds the bone's name and index in the model as well as the information to unpack the packed animation data. The index must be used to apply the animation to the correct bone in the model. The name can be used to validate that the animation is meant for a given model.

  - one 'bone translation data' chunk (type `0x1004`):
    ```
    uint16[3 * header.numFrames] translations
    ```
    This chunk holds an array of 16-bit unsigned integers that, in triples, define an array of packed translation vectors, one for each frame of the animation. The unpacked translation vector for frame _i_ can be calculated as:
    ```
    unpackedTranslation = header.translationOffset + float3(translations[i*3+0], translations[i*3+1], translations[i*3+2]) * header.translationScale
    ```
    This chunk can be omitted if the bone does not translate during the animation. If omitted, the translation for the entire animation should be set to the `translationOffset` in the 1003h chunk.

  - one 'bone scale data' chunk (type `0x1005`):
    ```
    uint16[3 * header.numFrames] scales
    ```
    This chunk holds an array of 16-bit unsigned integers that, in triples, define an array of packed scale vectors, one for each frame of the animation. The unpacked scale vector for frame _i_ can be calculated as:
    ```
    unpackedScale = header.scaleOffset + float3(scales[i*3+0], scales[i*3+1], scales[i*3+2]) * header.scaleScale
    ```
    This chunk can be omitted if the bone does not scale during the animation. If omitted, the scale for the entire animation should be set to the `scaleOffset` in the 1003h chunk.

  - one 'bone rotation data' chunk (type `0x1006`):
    ```
    int16[4 * header.numFrames] rotations
    ```
    This chunk holds an array of 16-bit signed integers that, in quadruples, define an array of packed rotation quaternions, one for each frame of the animation. The unpacked rotation quaternion for frame _i_ can be calculated as:
    ```
    unpackedRotation = quaternion(rotations[i*4+0] / 32767.0, rotations[i*4+1] / 32767.0, rotations[i*4+2] / 32767.0, rotations[i*4+3] / 32767.0)
    ```
    Note: This chunk is always present and contains either `header.numFrames` quaternions, or a single quaternion. In the former case, the bone has a different rotation for each frame, in the latter case, the bone's rotation does not change during the animation and should be set to the specified rotation.

  - one 'bone visibility data' chunk (type `0x1007`):
    ```
    uint8[(header.numFrames + 7) / 8] visibility
    ```
    This chunk holds an array of visibility bits (stored in bytes), one for each frame of the animation. Each bit indicates if the bone (and attached objects) is visible at its frame. The visibility bit for frame _i_ can be calculated as:
    ```
    unpackedVisibility = (visibility[i / 8] >> (i % 8)) & 1
    ```
    This chunk is optional. If missing, the bone's visibility from the model is used.

  - one 'bone step data' chunk (type `0x1008`):
    ```
    uint8[(header.numFrames + 7) / 8] stepped
    ```
    This chunk holds an array of 'step' bits (stored in bytes), one for each frame of the animation. Each bit indicates if the bone animation steps from its frame to the next. Such a step is a discontinuous change in the animation properties. This can be useful to decide whether or not to interpolate between adjacent frames. The 'step' bit for frame _i_ can be calculated as:
    ```
    unpackedStep = (stepped[i / 8] >> (i % 8)) & 1
    ```
    This chunk is optional.

## Format #2

This format is an optimization of format #1. It contains the same fundamental information, but the scale, translation and rotation data for all bones are combined per frame to achieve better cache locality when reading the data into memory as a whole.

This format contains the following changes respective to format #1:
- Chunk type `0x1001` is extended and is now:
  ```
  01: dword numFrames
  02: float fps
  03: dword numBones
  11: dword rotationFrameSize     # new
  12: dword translationFrameSize  # new
  13: dword scaleFrameSize        # new
  ```
  The three new mini-chunks define the number of 16-bit integers in each frame for the rotation, translation and scale data, respectively.
  
- Chunk types `0x1004`, `0x1005`, and `0x1006` are removed.

- Chunk type `0x1003` is extended and is now:
  ```
  04: string name
  05: uint32 index
  06: float3 translationOffset
  07: float3 translationScale
  08: float3 scaleOffset
  09: float3 scaleScale
  10: uint32 unknown
  14: uint32 translationIndex   # new
  15: uint32 scaleIndex         # new
  16: uint32 rotationIndex      # new
  17: int16[4] defaultRotation  # new
  ```
  The four new mini-chunks at the end hold the index of the bone's translation, scale and rotation data in each frame's block in the new `0x1009`, `0x100a` and `0x100b` chunks, respectively.
  If any index is `-1`, this bone does not have that data and the default value (`translationOffset` for translation, `scaleOffset` for scale, `defaultRotation` for rotation) should be used for all frames instead.

- A new 'animation rotation data' chunk (type `0x1009`) is added as a child of `0x1000`:
  ```
  int16[4 * header.numFrames * header.translationFrameSize] rotations
  ```
  This chunk holds an array of packed quaternions for each frame for each animated bone. It is effectively an array of one block per frame, where each block contains the rotation data for all bones in a frame. A bone's `rotationIndex` in the `0x1003` chunk points to the start of the bone's data into a frame's block. Thus, the unpacked rotation quaternion for frame _i_ and bone _b_ can be calculated as:
  ```
  start = i * header.rotationFrameSize + bones[b].rotationIndex
  unpackedRotation = quaternion(rotations[start*4+0] / 32767.0, rotations[start*4+1] / 32767.0, rotations[start*4+2] / 32767.0, rotations[start*4+3] / 32767.0)
  ```

- A new 'animation translation data' chunk (type `0x100a`) is added as a child of `0x1000`:
  ```
  int16[3 * header.numFrames * header.translationFrameSize] translations
  ```
  This chunk holds an array of packed vectors for each frame for each animated bone. It is effectively an array of one block per frame, where each block contains the translation data for all bones in a frame. A bone's `translationIndex` in the `0x1003` chunk points to the start of the bone's data into a frame's block. Thus, the unpacked translation vector for frame _i_ and bone _b_ can be calculated as:
  ```
  start = i * header.translationFrameSize + bones[b].translationIndex
  unpackedTranslation = header.translationOffset + float3(translations[start*3+0], translations[start*3+1], translations[start*3+2]) * header.translationScale
  ```

- A new 'animation scale data' chunk (type `0x100b`) is added as a child of `0x1000`:
  ```
  int16[3 * header.numFrames * header.scaleFrameSize] scales
  ```
  This chunk holds an array of packed vectors for each frame for each animated bone. It is effectively an array of one block per frame, where each block contains the scale data for all bones in a frame. A bone's `scaleIndex` in the `0x1003` chunk points to the start of the bone's data into a frame's block. Thus, the unpacked scale vector for frame _i_ and bone _b_ can be calculated as:
  ```
  start = i * header.scaleFrameSize + bones[b].scaleIndex
  unpackedScale = header.scaleOffset + float3(scales[start*3+0], scales[start*3+1], scales[start*3+2]) * header.scaleScale
  ```

## Game support notes
Star Wars: Empire at War uses the first animation format. Its expansion, Forces of Corruption, and later games, use the second format.

