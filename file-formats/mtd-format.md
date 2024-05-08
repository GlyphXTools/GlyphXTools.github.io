# Mega-Texture Directory File Format

Mega-Texture Directories (extension: .MTD) in Petroglyph's are used to store a list of _<image name, image info>_ pairs, essentially a directory of images. The image info is a set of (X,Y) coordinates and Width x Height. This information describes a rectangle in another image file, where this image is located.

By this method, many small images can be packed into one large image (the Mega-Texture) and located again through the accompanying directory. Doing this saves overhead creating every small file.

## Format
Each Mega-Texture Directory begins with a header, followed by the image index table.

```
Header:
  +0000h  count      uint32      ; Number of records in the Images Index Table

Image Index Table record:
  +0000h  name       64 bytes    ; ASCII name of the image, zero-padded if less than 64 bytes long.
  +0040h  x          uint32      ; X position in the Mega-Texture of the image, in pixels
  +0048h  y          uint32      ; Y position in the Mega-Texture of the image, in pixels
  +004Ch  width      uint32      ; Width of the image, in pixels
  +0050h  height     uint32      ; Height of the image, in pixels
  +0054h  alpha      uint8       ; 1 if this image uses the Alpha Channel
```
  
The format is pretty straightforward, and the alpha boolean for every image is pretty useless information.

## Border extension
The following information is critical if you plan on creating MTD files!

Because the Mega-Texture is created and used a texture on 3D primitives (although screen-align), round-off sampling errors can still occur, potentially showing a single line of pixels from another image to either side of the image.

To prevent this, all images have their border extended by one pixel when they are packed in the Mega-Texture. However, the Image Index indicates the original area. This means that when packing additional images into a Mega-Texture, the 1-pixel area around existing images should be treated as reserved and filled with the border for new images.