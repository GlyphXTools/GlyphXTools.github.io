# Lua File Format

[Lua](http://www.lua.org/) is a programming language that is very suited for plug-in programming in games. Lua source files are compiled by the game or a lua compiler and then used by the game to perform campaign scripting, AI scripting, etc.

The compiled Lua files, however, are of a slightly different format than standard Lua files.

## Format #1
This format is used in the games Empire at War and Forces of Corruption.

Each Petroglyph Lua object file is basically a normal [Lua 5.0 object file](http://www.lua.org/source/5.0/src_lundump.c.html#luaU_undump) except for three differences:

* The signature for Lua 5.0 object files is "\033Lua". For Petroglyph's Lua object files this is "\033Lup"
* The byte that indicates the file version is 0x50 for the Lua 5.0 object files. Petroglyph's Lua object files use 0x51 (it really IS a 5.0 file though).
* In each function header, Petroglyph's Lua object files have an additional integer right after the "lineDefined" integer that can be ignored.

The additional integer seems to start at 1 for the first function in the file, and then increases by one for every other function, disregarding function nesting. This integer is supposedly used to allow for persistent Lua state during save-games.

## Format #2
This format is used in the game Universe at War.

Each Petroglyph Lua object file is basically a normal [Lua 5.1 object file](http://www.lua.org/source/5.1/lundump.c.html#luaU_undump) except for two differences:

* The format for a Lua 5.1 object files is 0. For Petroglyph's Lua object files this is 112 (ASCII for 'p')
* In each function header, Petroglyph's Lua object files have an additional integer right after the "lastLineDefined" integer that can be ignored.

The additional integer seems to start at 1 for the first function in the file, and then increases by one for every other function, disregarding function nesting. This integer is supposedly used to allow for persistent Lua state during save-games.

## Converter
To allow Petroglyph's Lua object files to be handled by any existing Lua 5 object file program, [luacvt](https://github.com/GlyphXTools/lua-converter) is a small program that can convert Petroglyph's Lua object files to normal Lua 5 object files and back.