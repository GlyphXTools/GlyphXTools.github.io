# Mega File Format

Mega Files (extension: .MEG) in Petroglyph's games are used to store the collection of game files. By packing these files into a single large file, the operating-system overhead of storing and opening each individual file is removed and it helps avoid fragmentation.

## Format #1
Each Mega File begins with a header, followed by the Filename Table, the File Table and finally, the file data. All fields are in little-endian format.

```
Header:
  +0000h  numFilenames  uint32   ; Number of filenames in the Filename Table
  +0004h  numFiles      uint32   ; Number of files in the File Table

Filename Table record:
  +0000h  length        uint16   ; Length of the filename, in characters
  +0004h  name          length   ; The ASCII filename

File Table record:
  +0000h  crc           uint32   ; CRC-32 of the filename
  +0004h  index         uint32   ; Index of this record in the table
  +0008h  size          uint32   ; Size of the file, in bytes
  +000Ch  start         uint32   ; Start of the file, in bytes , from the start of the Mega File
  +0010h  name          uint32   ; Index in the Filename Table of the filename
```

## Format #2
This format is an extension on format #1 by adding extra fields in the header.

Each Mega File begins with a header, followed by the Filename Table, the File Table and finally, the file data. All fields are in little-endian format.

```
Header:
  +0000h  id1           uint32   ; Unknown field, always 0xFFFFFFFF
  +0004h  id2           uint32   ; Unknown field, always 0x3F7D70A4
  +0008h  dataStart     uint32   ; Offset in file of start of data
  +000Ch  numFilenames  uint32   ; Number of filenames in the Filename Table
  +0010h  numFiles      uint32   ; Number of files in the File Table

Filename Table record:
  +0000h  length        uint16   ; Length of the filename, in characters
  +0004h  name          length   ; The ASCII filename

File Table record:
  +0000h  crc           uint32   ; CRC-32 of the filename
  +0004h  index         uint32   ; Index of this record in the table
  +0008h  size          uint32   ; Size of the file, in bytes
  +000Ch  start         uint32   ; Start of the file, in bytes, from the start of the Mega File
  +0010h  name          uint32   ; Index in the Filename Table of the filename
```

## Format #3
This format is an extension on format #2 by adding extra fields in the header and reformatting the file table record.

Each Mega File begins with a header, followed by the Filename Table, the File Table and finally, the file data. All fields are in little-endian format.

Mega Files in this format can be encrypted. This is indicated by the flags field in the header being 0x8FFFFFFF. Encrypted MegaFiles encrypt the following fragments:

* The Filename Table. This is encrypted as single blob of data.
* Each File Table record. Each record is individually encrypted, save for the flags field. If encrypted, a record's flags field will be 1, otherwise 0. Note that if the record is encrypted, its contents (after the flags field) is padded to 32 bytes.
* Every individual file's data is encrypted as a single blob of data. Note that each blob's size is rounded up to the next multiple of the AES block size.
Each encrypted fragment is encrypted using 128-bit AES in CBC mode, using the same key and IV. There is a section below listing the keys and IVs used by various games.

```        
Header:
  +0000h  flags         uint32   ; Flags field, 0xFFFFFFFF or 0x8FFFFFFF (unencrypted file, encrypted file resp.)
  +0004h  id            uint32   ; Unknown field, always 0x3F7D70A4
  +0008h  dataStart     uint32   ; Offset in file of start of data
  +000Ch  numFilenames  uint32   ; Number of filenames in the Filename Table
  +0010h  numFiles      uint32   ; Number of files in the File Table
  +0014h  filenamesSize uint32   ; Size, in bytes, of the Filename Table

Filename Table record:
  +0000h  length        uint16   ; Length of the filename, in characters
  +0004h  name          length   ; The ASCII filename

File Table record:
  +0000h  flags         uint16   ; 1 if entry is encrypted, 0 otherwise.
  +0002h  crc           uint32   ; CRC-32 of the filename
  +0006h  index         uint32   ; Index of this record in the table
  +000Ah  size          uint32   ; Size of the file, in bytes
  +000Eh  start         uint32   ; Start of the file, in bytes, from the start of the Mega File
  +0012h  name          uint16   ; Index in the Filename Table of the filename
  +0014h  (padding)     14 bytes ; If this entry is encrypted, the record data (everything after the flags) is padded to 32 bytes
```

## Notes
* The number of files and number of filenames are always equal (so far).
* The filenames are NOT zero-terminated.
* The File Table is sorted on the CRC (in ascending order).

## Using multiple Mega Files
It is common or just possible for Petroglyph's games to have multiple Mega Files, and even Mega Files for user mods. All used Mega Files are listed in MegaFiles.xml in a game directory. Petroglyph's games read this file and, load all Mega Files and merge their File Table to create one Master File Table. If a file occurs in multiple Mega Files, the file in the Mega File listed last in MegaFiles.xml will be used.

## Encryption/Decryption Keys
Some games using the Mega File format use the encryption capability offered with version 3. To be able to read and write such files, the AES key and initial vector must be known. Listed below are the known keys and initial vectors for various games.

### Grey Goo:
* Key: `63223401b27efb502ec657b134a92561` (or MD5("{CAF1CCE6-CC1D-40CE-AF2D-792BC446FD87}{83733AEE-CB3D-426E-BE9C-8CCB92E35B75}"))
* IV: `954fd996438b8fd035a5c7ffb6f6066b` (or MD5("Goo"))

### 8-Bit Armies:
* Key: `1CB3FAFE676920DC6B12E15B232DAD6D` (or MD5("{CAF8CCE6-CC1D-40CE-AF2D-792BC4987D87}{83733AEE-CB3D-426E-B33C-8CC224E35B75}"))
* IV: `6CF6B9ABE7878212F5DFAEE6CF8A1E18` (or MD5("SimpleRTS"))

### The Great War: Western Front:
* Key: `0251255BA84F03BBA7780225E37A954C` (or MD5("{88CCC849-676F-4E34-BDC2-A0C99C2D5F80}{EA1CB2E7-BFC4-4B44-97AA-52B7774F0713}"))
* IV: `6CF6B9ABE7878212F5DFAEE6CF8A1E18` (or MD5("SimpleRTS"))