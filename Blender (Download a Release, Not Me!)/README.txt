Mjolnir Forge Editor - map palette & model reader patch
=========================================================

What it does
------------
When you press File > Import > Forge Objects, Mjolnir now:
  1. Finds the .map file MCC currently has loaded (it asks Windows which .map files MCC has open,
     so modded maps in Workshop/mod folders are found automatically).
  2. Reads that map's Forge palette, so every object ID gets the right name, even on modded maps
     whose palette is changed or extended (e.g. the green screen mod).
  3. For objects with no mesh in forge.blend, builds the mesh from the .map's own render model,
     including the parts the game bolts on: model-variant attachments (turrets, guns) and a
     vehicle's own weapons. The palette item's variant (e.g. Warthog gauss/rocket) is respected.
     New meshes go in the "Extracted Models" collection (Props scene) and also show up under
     Add > Forge Object, so you can place them from Blender too.
  4. Gives those models their textures from the .map: base colour (with the shader's tint),
     normal map and self-illumination. Images are packed into the .blend, so each texture is
     decoded once; the first import of a textured vehicle takes a few seconds longer.
  5. Shows the loaded map behind your forge objects. The first time you import on a map (Forge World
     included), its textured level geometry and sky are read from the .map into their own scene. Your
     working scene's unselectable "Map" backdrop is then pointed at that map, and forge objects are
     imported into the working scene. (forge.blend no longer ships a hand-made Forge World: it's built
     from forge_halo.map like every other map, which keeps the download about 19 MB smaller.)

  6. Add > Forge Object... opens a search with a preview: type to filter, click an item to see a
     large thumbnail of it, then Enter or Add places it at the 3D cursor. Stock items come with
     thumbnails; newly extracted models get one rendered automatically (about a second each).

  7. Makes it look closer to the game (Game Lighting, on by default): the map's sky model is added to
     its map scene (unlit, so it looks the way the game draws it), the scene switches to EEVEE with a
     sun that casts shadows, ambient light tinted by the sky's colour, ambient occlusion, and the
     viewport switches to Material Preview with scene lights. Done once per scene, so your own lighting
     tweaks are kept; the sun button next to the setting re-applies it (and rebuilds the sky).
     Materials now use more of Halo's shader: detail maps, terrain blending of up to four layers
     (with their normal maps), and specular strength / roughness from the shader's own values.
     Baked Lighting (experimental, on by default): the level is lit like the game from the map's own
     lightmaps: baked indirect/sky light plus the sun, with the game's baked sun shadows. The sun's
     direction is worked out from those baked shadows, so Blender's sun (which lights forge objects)
     matches the map, and forge objects cast real-time shadows onto the level. The importer first checks
     that the lightmap lines up with the geometry and falls back to plain sun lighting if it doesn't.
     Rotate the "Sun" and press Set Up Game Lighting to change the sun direction yourself.

Copying forge objects between .blend files (Ctrl+C in one Blender, Ctrl+V in another) works: pasted
objects are pointed back at the receiving file's own models, and the duplicate models Blender brings
along (which used to show up as loose parts at the world origin) are removed automatically. Models
the receiving file doesn't have (e.g. extracted on another map) are kept, under Extracted Models.

Models extracted by an earlier version of this patch are rebuilt automatically on the next import.

If a model comes in facing the wrong way (can happen with ported/modded models), select one copy and
use right-click > Rotate Model 90 deg. It turns the shared model, so every copy turns; repeat as needed.

Export uses the same palette, so objects are written back with the correct IDs for the modded map.
Objects pasted in from another map are matched to this map by name, or for extracted models by the
object's tag. Before writing to MCC, export leaves out (and lists in its message and the system
console) any object the loaded map can't hold, since those can crash the game: a type that isn't in
this map's palette, more copies of an item than the map allows, or a position far outside the map.

If anything fails, it falls back to the original behaviour (built-in tables), and says why in the
import message and the Forge Scene panel.

Install
-------
Extract forge.blend and ForgeBridge.dll into the same folder (the DLL must sit next to the .blend).
Open forge.blend and allow scripts to run, as before. The DLL itself is unchanged.

  forge.blend              patched forge.py + the mjolnir_mapreader.py text block embedded
  ForgeBridge.dll          the standard Mjolnir DLL (reads/writes forge objects in MCC's memory)
  forge.py                 the patched script on its own (paste into your own forge.blend instead,
                           together with mjolnir_mapreader.py as a text block or file next to it)
  forge.patch              the changes relative to the original forge.py
  mjolnir_mapreader.py     the reader module, also usable from a command line (see below)

Settings  (Properties editor > Scene tab > Forge Scene > Map Palette)
--------
  Read Palette from Map    on by default
  Extract Missing Models   on by default
  Extract Textures         on by default (turn off for faster, untextured models)
  Build Map Scene          on by default (off = no map geometry is built or switched)
Lighting  (same panel, "Lighting" box)
  Game Lighting            on by default: sky + EEVEE lighting on import
  Baked Lighting           on by default: level lit by the map's baked lightmaps (experimental)
  Baked Brightness         1 = automatic exposure
  Set Up Game Lighting     applies the above now (rebuilds the level if Baked Lighting was switched)
  Render Thumbnails        on by default (previews of newly extracted models for the search)
  Render Missing Thumbnails   renders previews for any palette item that has none
  Map File                 leave empty for auto-detect; set it if the wrong .map gets picked
  Mod Folders              extra folders to search (separated by ;) if auto-detect can't list MCC's files
  Extract All Palette Models  builds meshes for every palette item missing from forge.blend
                              (works without MCC running if Map File is set)

Checking a map without Blender
------------------------------
With any Python 3 install:
    python mjolnir_mapreader.py "C:\path\to\forge_halo.map"
prints every palette item with its forge ID and tag path.
    python mjolnir_mapreader.py "C:\path\to\forge_halo.map" --obj out_folder
also exports every palette model as .obj. Add  --textures tex_folder  to save their textures as
.png and print each shader's texture slots, tint colours and options. If shared.map isn't next to the map, add
    --shared "C:\...\Halo The Master Chief Collection\haloreach\maps"
    python mjolnir_mapreader.py "C:\path\to\70_boneyard.map" --bsp level_folder
exports the map's level geometry as .obj (default Blender OBJ import settings work).
    python mjolnir_mapreader.py "C:\path\to\70_boneyard.map" --lighting probe_folder
prints the map's sky, level shader settings and baked lightmap textures (and saves what it can
decode). Decoding textures from the command line needs numpy:  pip install numpy
Share that output to help add the game's baked lighting.
Run it with no map path while MCC is running a map to test the auto-detection.

Limits
------
- Materials approximate Halo's shaders: base x detail maps x albedo tint, terrain layer blending,
  normal maps, specular strength/roughness, self-illumination, alpha test/blend. Team colour
  (change colour), cube-map reflections and detail normal maps aren't reproduced. Additive shaders
  (glows) show as emission.
- Baked lighting doesn't use the lightmap's directional detail for the indirect light, so bump detail
  in shade is flatter than in game; specular highlights aren't drawn on baked surfaces. Real-time
  object shadows on the level need EEVEE (Cycles shows the baked shadows only).
- Cube maps and 3D textures are skipped. The level's placed scenery/machines aren't built yet,
  only the level geometry and sky.
- Models use the variant's permutations, else the "default"/first permutation of each region.
- Attachments are placed in the model's default pose (turrets point straight ahead).
- If a model can't be decoded, a wireframe box of its bounding box is used instead.
- File format code is ported from Reclaimer and Assembly; it was tested against synthetic maps,
  not yet a real MCC map. If something is off, run the command-line check above and share the output.
