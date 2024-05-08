# Creating an Empire at War Mod

So, you want to mod Empire at War, do you? Fortunately, this is very easy.

## Changing files
All of Petroglyph's games are data-driven. This means that it will read data files from the disk, which contain references to other data files, so it will load those, and so on until it has all the information it needs. Initially, all those data files are packed into Mega Files. Think of them as ZIP files, but without the compression. You can extract the files in them with FinalBIG.

However, after editing a file you do not have to put them back in its MegaFile, because the game looks at the files in its directory first! This means that you can just put the modified file into the game directory (with the same subpath as found in the Mega File) and the game will use it instead.

For example, consider that you want to change Data\XML\Audio.xml for Universe at War: Earth Assault. This file is found in Config.meg (C:\Program Files\Sega\Universe at War Earth Assault\Data\Config.meg), so you extract it somewhere and edit it as desired. Then you can save it as C:\Program Files\Sega\Universe at War Earth Assault\Data\XML\Audio.xml and the game will use it instead.

## MODPATHs
Unfortunately, this changes the game anytime you want to play it. This also means you cannot connect to players unless they are running with the same modified files. And maybe you'd like to have multiple mods. Or maybe you'd like to play the unmodded game once in a while. Moving or renaming files and directories all the time is a bit of a hassle, isn't it?

Fortunately, there's a better way. The game executable accepts an argument that allows you specify a path where it should also look for files. This path is called the MODPATH. By convention, mods go into the Mods directory in the game directory.

Using the example above, your modified Audio.xml would go into C:\Program Files\Sega\Universe at War Earth Assault\Mods\My_Mod\Data\XML\Audio.xml, where you can name your mod whatever you like (My_Mod in this case). Now all that's left is starting the game with the correct MODPATH argument. You'll want the command line to be (for Universe at War, for instance): "LaunchUAW.exe MODPATH=Mods\My_Mod" (without the quotes). I recommend creating a shortcut to the game executable (e.g., LaunchUAW.exe) and then editing the shortcut to add "MODPATH=Mods\My_Mod" to it. That shortcut will then start your mod.

## Conclusion
To summarize, if the game wants to load file X, it will look for X in the following places, in this order:

1. The `MODPAT`H directory, if specified.
1. The directory where the game executable resides.
1. All Mega Files specified in `Data\MegaFiles.xml`

## File formats
Now that you know where to place edited files, all that remains is knowing what types of files are used in Petroglyph's games and how they can be created or edited. Behold:

| Extension | Directory | Contents | Note |
| - | - | - | - |
| ALO | Data\Art\Models | Objects | Use the Alo Exporter to create ALO and ALA files from 3D Studio Max. Use the Alo Importer to import ALO and ALA files back into 3D Studio Max. Each ALA file is a single animation, its filename prefixed with the object name. |
| ALA | Data\Art\Models | Animations |
| ALO | Data\Art\Models | Particles | Use the Particle Editor to create or edit particles. |
| TED | Data\Art\Maps | Maps | Use the Map Editor to create or edit maps. |
| DAT | Data\Text | String files | DAT files contain all strings used in the game. Edit with the String Editor. |
| MTD | Data\Art\Textures | Mega Texture Directory | This file, combined with its similary-named TGA file form a Mega-Texture. They can be edited with the MTD Editor. |
| DDS | Data\Art\Textures | Textures | These files contain the textures used by everything in the game from objects to terrain. The game will try both extension when looking for these files. |
| TGA | Data\Art\Textures | Textures | (see above) |
| XML | Data\XML | XML files | The XML files contain most description of units, objects, effects, rules and other gameplay basics. Edit them with a simple text editor or specialized XML editor. |
| LUA | Data\Scripts | Lua Scripts | Lua Scripts control a lot of the gameplay logic, including AI and even the GUI. The Lua files that come with the game are compiled and should not be edited. The mod tools have the Lua sources which should be used instead. |
| WAV | Data\Audio\Speech | Speech | These are standard WAV files for Speech events. There are localized version for several languages. Edit with any WAV editor. |
| WAV | Data\Audio\SFX | Sound effects | These are standard mono WAV files for sound effects events. Edit with any WAV editor. |
| MP3 | Data\Audio\Music | Music | These are standard MP3 files for game music. Edit with any MP3 editor. |
| FX | Data\Art\Shaders | DirectX Shaders | FX files can be edited with any plain text editor such as notepad. |
| FXO | Data\Art\Shaders | DirectX Shaders | FXO files are compiled FX files and should not be edited. |