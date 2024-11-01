# Chunked File Format

## Introduction
Many files used in Petroglyph's games, from models and animations to maps and templates, have a hierarchical file structure. Regardless of the contents of the file, all these files share the same base structure: a tree, where each node has a header with type and size. This format allows games and tools to ignore chunks they do not understand or are not interested it. The chunked file format describes how this tree is stored in the files.

## Format
The basic element in the chunked file format is a chunk. A chunk represents a node in the file's tree. A chunk's contents can either be data, or other chunks (child chunks). At the top-level, each file can have multiple root chunks.

Each chunk is described via a header:

```
Header:
  +0000h  type      uint32      ; Type of the chunk
  +0004h  size      uint32      ; Size of the chunk, in bytes
```
The `type` field allows the program to know what the chunk should contain. The `size` field indicates the size of the body of this chunk, which immediately follows the header. This body consists either of data or other chunks (never both). To distinguish between these cases, the high bit (bit 31) of the size field is used. If it is set, the chunk contains child chunks. If it is clear, the chunk contains data. Thus, bit 31 must be ignored when you want to know the size of the chunk body.

Each chunk (and thus chunked file) begins with a chunk header. If the chunk body does not fill up the entire parent chunk or file, there is another chunk behind it.

### Mini-chunks
Sometimes, chunks contain very little data. To avoid the overhead of the chunk header, _mini-chunks_ can be used. This a format which is used on chunks that contain data (so bit 31 of the size is NOT set). Just like chunks that contain child chunks, these chunks are entirely chopped up into child mini-chunks. However, each mini-chunk has a smaller header:

```
Mini-chunk Header:
  +0000h  type      byte      ; Type of the mini-chunk
  +0001h  size      byte      ; Size of the mini-chunk, in bytes
```
Since mini-chunks cannot contain children, all eight bits of the size field are used for the size. Note that there is no built-in way to distinguish between chunks with data and chunks with mini-chunks. The file format defined on top of the Chunked file format must define which chunk types contain mini-chunks.

## Notes
The meaning of the chunk type depends on the file type, there is no 'global list' of chunk types. 

## Chunk Viewer
You can use the [Chunk Viewer](https://github.com/GlyphXTools/chunk-viewer) to explore the tree structure of chunked files. Note that since it cannot know whether a chunk contains mini-chunks or not, it makes educated guesses based on the data in the chunk. Fortunately, it's mostly right.