# Jester ([docs](https://aoughwl.com))

[Unity](https://unity.com), turned into a game you mod while it is running.

Roblox and s&box let you build games on someone else's engine, in their
sandbox, shipped through their platform. Jester takes the other half of the
idea: a real engine, a real compiled language, no platform, and no built-in
game to work around. The host boots, loads a shell mod, and the shell picks a
modpack. Player, weapons, world, inventory, the menu you just used: all mods,
all replaceable, all swappable without stopping the game.

|  | Roblox | s&box | Jester |
| --- | --- | --- | --- |
| Language | sandboxed Lua | C# | aowlmony, statically typed, compiled |
| Iteration | restart the place | recompile, reload | edit while running, state survives |
| Built-in game | theirs | theirs | none; the menu is a mod |
| Cross-mod coupling | direct references | direct references | catalogs, no imports |
| Engine access | sandboxed | their API | typed API, with a raw escape hatch |
| Assets | their store | their pipeline | read them out of games you own |
| Distribution | their platform, revenue cut | their platform | one build, your releases |

Mods are written in [aowlmony](https://aoughwl.com): a from-scratch clone of
the unreleased Nim 3.0, except finished, and with an interpreter. Source is
compiled to a typed intermediate form and executed by `aowli`.

```
proc start() =
  create("weapons", "item")

proc update() =
  if pressed(Space):
    spawn("cube", at(0, 3, 0))
```

Three hacks hold it up.

## Hack 1: the host is a very small Unity game

Jester is not an engine fork, a custom renderer, or a Unity plugin. It is a
Unity project of a few thousand lines that draws nothing and plays nothing. It
loads an interpreter, hands it a mod, and answers the calls that come back.

The boundary is 258 host calls. That is the entire contract between a mod and
Unity, and the C# dispatch table and the mod-side declarations are the same
258 names, and the two lists are checked against each other rather than merely
counted. Gameplay never crosses into C#; C# never knows what game is running.

Because the host is small, it can be frozen. Ship it once, and every game after
that is content.

## Hack 2: reload the code, keep the data

`aowli` separates a module's code from its data. Rebuilding a mod replaces the
code and leaves the interpreter's globals where they are, so `update()` keeps
running against the same values it had a frame earlier. The host watches
artifact timestamps and swaps on change: no restart, no reconnect, no reload
prompt.

State that must outlive the process goes through `remember(key, default)`,
which reads from a host-side store rather than interpreter memory. That store
is flushed before every modpack switch and teardown, so it survives a crash, a
quit, or swapping the whole game out from under itself.

A mod that throws is quarantined, named, and left loaded, so fixing the file
brings it back without touching the session.

## Hack 3: mod games that were never meant to be modded

Every game ships its content in some container: an archive format, a mesh
format, a texture format. Read the container and the content is yours to run
somewhere else.

A mod claims a file extension or a URI scheme. When something asks for a model
nobody recognises, that mod is handed the bytes and calls
`beginMesh`/`addVertex`/`addTriangle`/`finishMesh`. That is the entire
mechanism, and it is why the readers that ship — Source VPK/MDL/VTF, Minecraft
jars and resource packs, Tarkov bundles — are ordinary mods you can delete,
replace, or ignore.

What it buys is bigger than a file format. A game's own modding surface is
whatever its developers chose to expose, and it is usually a fraction of what
is on disk. Read the container yourself and the ceiling is gone: the weapons,
the animations, the environments and the UI all become catalog entries in an
engine that hot-reloads, has no scripting sandbox, and does not care which game
they came from. You can build in a universe you like, with tools its own
authors never shipped.

Reading happens in-game, on the player's machine, from a copy they installed.
Jester ships no third-party content and redistributes none: multiplayer
replicates catalog references and a content signature rather than geometry, so
every peer rebuilds from the copy they own. Each game's licence governs what
you may extract and what you may do with it afterwards.

## Two ways down to the engine

Most engines make you pick: a safe high-level API that cannot do the thing you
need, or raw access that you will regret. Jester ships both, at the same time,
to the same mod.

The floor is direct: `unity_get`, `unity_set`, `unity_call` and
`component_add` reflect into UnityEngine types, so a mod can reach an engine
feature the SDK never wrapped without waiting for anyone. Above it sits a typed
API — vectors, colours, entities, input, drawing, meshes — that is what you
should actually write, and above that optional template sugar. Three layers,
each one built out of the one below, none of them privileged.

Networking is the worked example. Nothing stops a mod opening sockets and
writing its own protocol; the floor is right there. You should not. The host
already mints peer identity on the server, decides placement authority per
frame rather than by per-channel permissions, and replicates parts as catalog
references plus a signature rather than as geometry, so peers rebuild from
content they already have. It has been run with real processes over real
sockets at 100k parts with zero events dropped. The escape hatch exists so you
are never blocked; the layer above exists so you rarely want it.

## Catalogs, and what they cost

Mods cannot import each other. They publish into catalogs: one mod declares a
kind, another creates a catalog of that kind, a third fills it, and every entry
records its contributor. A weapons mod creates `weapons`; a Source reader adds
entries to it; an inventory mod iterates it without knowing either exists;
removing the reader removes exactly its entries.

The benefit is real — no load order, no registry, no central file to edit, and
one mod cannot break another by changing a signature, because there are no
signatures to change.

The costs are also real, and worth knowing before you build on it:

- **Data, not behaviour.** Catalogs carry values. A mod cannot call another
  mod's function, so a library mod publishes facts rather than an API, and
  anything genuinely shared ends up in the SDK instead.
- **One value type per catalog.** A definition with six fields is six
  catalogs, keyed alike by convention. It works and it is ugly.
- **Stringly typed.** Ids and kinds are text, agreed by convention across mods
  that never see each other's source. Nothing catches a typo at compile time.
- **Absence is normal.** With no load order, a catalog you read may not exist
  yet, so every reader has to be written to tolerate nothing being there.

It is a deliberate trade: coupling that cannot break, bought with coupling that
cannot be checked.

## Distribution

One host build, shipped once. Content updates arrive as mods, verified by hash,
applied atomically, and picked up by the existing reload path without a
restart. The host self-updates by staged replacement when it has to, which is
rare by design. Releases go through GitHub Releases.

## Get it

Builds are published under [Releases](https://github.com/savannt/jester-release/releases).
Download, unzip, run. The game updates itself from there afterwards.

## Docs

Full documentation, the mod API and the aowlmony language reference:
**[aoughwl.com/docs/jester](https://aoughwl.com/docs/jester)**

## This repository

This is the distribution repo: the build and the readme. The engine source is
developed separately. Issues and discussion belong here.
