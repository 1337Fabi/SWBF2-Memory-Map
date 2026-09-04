# Star Wars Battlefront II Classic — Memory Map

Executable addresses are relative virtual addresses (RVAs) from the runtime
`BattlefrontII.exe` base. Structure offsets are relative to the named object.

## Module globals

| RVA | Type | Function |
|---:|---|---|
| `0x1A30324` | `Game*` | Global game object |
| `0x1B9AE5C` | `SpawnDisplay*` | Global class-selection display object |
| `0x1BAC0E0` | inline `wchar_t[]` | Local-player name string |
| `0x1B3A5D4` | inline `wchar_t[]` | Alternate local-player name string |

## Internal functions and decisions

| RVA | Function |
|---:|---|
| `0x0005E400` | Geometry trace: `bool __fastcall TraceLine(float*, float*, int, int, int)` |
| `0x0024E510` | Character-list creation routine |
| `0x00029A20` | Resolves the player/display argument used before class-display activation |
| `0x00029DD0` | Lower class-display activation routine |
| `0x0002A670` | Class-display activation wrapper; prepares state and calls the lower activation routine |
| `0x000EFC84` | Tests control-point eligibility bit `0x02` at context offset `0x472` |
| `0x000EFC8B` | Control-point class-selection rejection branch |

`TraceLine` entity argument:

```text
Entity* - 0x240
```

## `Game` — size `0x140`

| Offset | Type | Function |
|---:|---|---|
| `0x0000` | embedded `CameraManager` | Camera-manager base object |
| `0x0010` | `Character (*)[64]` | Character array |

Character address:

```text
Game->characterList + index * 0x1B0
index range: 0..63
```

## `Character` — size/stride `0x1B0`

| Offset | Type | Function |
|---:|---|---|
| `0x0030` | inline `wchar_t[]` | Character-name start |
| `0x0134` | `int32_t` | Numeric team ID (`1` or `2`) |
| `0x0138` | `Team*` | Team object |
| `0x013C` | `int32_t` | Current class index; first class is `0` |
| `0x0140` | `EntityClass*` | Current class definition |
| `0x0148` | `Entity*` | Live entity |
| `0x014C` | `Vehicle*` | Occupied vehicle; null while the character is on foot |

## World-object intrusive list

Soldiers and vehicles share an intrusive circular world-object list.

```text
object + 0x04 -> embedded list node
object + 0x08 -> next object's node pointer
next object    = *(object + 0x08) - 0x04
```

Begin traversal from a validated live `Character + 0x148` entity. Stop on a repeated, null, unreadable, or invalid node.

## Vehicle objects

### Common vehicle fields

| Offset | Type | Function |
|---:|---|---|
| `0x0000` | vtable pointer | Vehicle category discriminator |
| `0x0004` | list node | Embedded world-object list node |
| `0x0008` | pointer | Next world-object node |
| `0x0148` | `Vec3` | Vehicle world position |

### Vehicle categories

Vtable values are RVAs relative to the runtime `BattlefrontII.exe` base.

| Vtable RVA | Category | Class pointer offset |
|---:|---|---:|
| `0x0039BAD4` | Hover | `Vehicle + 0x024C` |
| `0x0039DD24` | Walker | `Vehicle + 0x0220` |
| `0x0039B3CC` | Flyer | `Vehicle + 0x03EC` |

Vehicle display name:

```text
vehicleClass = *(Vehicle + categoryClassOffset)
name         = *(wchar_t**)(vehicleClass + 0x20)
```

Vehicle occupant/team lookup:

```text
for each Character in Game->characterList:
    if Character + 0x14C == Vehicle*:
        Character + 0x134 -> occupant team ID
```

A null `Character + 0x14C` means that character is not occupying a vehicle. A vehicle with no matching character is empty.

### Stationary turret

Stationary turrets use a distinct runtime type and the same occupant link as
vehicles.

| Offset or RVA | Type | Function |
|---:|---|---|
| `0x003AA650` | vtable RVA | `Turret` runtime-type discriminator |
| `Turret - 0x0004` | `TurretClass*` | Turret class-definition pointer |
| `Turret + 0x0148` | `Vec3` | Turret world position |
| `0x003AA720` | vtable RVA | `TurretClass` runtime-type discriminator |

Turret occupant/team lookup:

```text
for each Character in Game->characterList:
    if Character + 0x14C == Turret*:
        Character + 0x134 -> occupant team ID
```

A turret with no matching character is empty. Team ownership for an empty
turret is not established by the occupant link.

## CTF flag objects

| Offset or RVA | Type | Function |
|---:|---|---|
| `0x0039CF2C` | vtable RVA | `FlagItem` runtime-type discriminator |
| `FlagItem + 0x00CC` | `Character*` | Current flag carrier; null while not carried |
| `FlagItem + 0x0148` | `Vec3` | Flag world position |

Carrier lookup:

```text
carrier = *(Character**)(FlagItem + 0xCC)
```

While a flag is carried, the reverse attachment is available at
`carrier + 0x14C` and points to the same `FlagItem`. Test the attached object's
vtable against RVA `0x0039CF2C` to distinguish a carried flag from a vehicle or
another attachment.

## `Team`

| Offset | Type | Function |
|---:|---|---|
| `0x0008` | `int32_t` | Embedded numeric team ID |
| `0x000C` | `wchar_t*` | Team name |
| `0x003C` | `int32_t` | Team size |
| `0x0780 + classIndex * 4` | `EntityClass*` | Faction-specific class-definition array |
| `0x0794` | `void*` | Team-specific class-menu context |

## `EntityClass`

| Offset | Type | Function |
|---:|---|---|
| `0x0020` | `wchar_t*` | Entity/class name |
| `0x0164` | `void*` | Class-specific UI context |

## `Entity` — size `0xB28`

| Offset | Type | Function |
|---:|---|---|
| `0x00C8` | `PlayerController*` | Player controller |
| `0x0148` | `Vec3` | Eye position |
| `0x0154` | `Vec3` | Native aim-trace endpoint in world space |
| `0x0164` | `Entity*` | Entity intersected by the native reticle trace; null when no entity is hit |
| `0x0168` | `uint32_t` | Reticle-hit type/value; observed as `0x88` for an enemy entity |
| `0x0174` | `Entity*` | Mirrored native reticle-target entity pointer |
| `0x0178` | `uint32_t` | Mirrored reticle-hit type/value |
| `0x0200` | `EntityClass*` | Entity class |
| `0x029C` | `Vec3` | Velocity |
| `0x02A8` | `float` | Body/Y rotation component |
| `0x02D8` | `Aimer*` | Aimer object |
| `0x02E4` | `Stats*` | Statistics and world-position object |
| `0x04E0` | `Weapon*` | Primary weapon |
| `0x04E4` | `Weapon*` | Secondary weapon |
| `0x04E8` | `Weapon*` | Utility weapon |
| `0x0500` | `uint8_t` | Selected weapon-slot index (`0` = primary, `1` = secondary) |
| `0x0504` | `int32_t` | In-air state |
| `0x0508` | `int32_t` | On-ground state |
| `0x0510` | `void*` | High-resolution pose workspace |
| `0x078C` | `GameModel*` | Game-model object |
| `0x0790` | `SoldierAnimatorLowRes*` | Optional low-resolution animator |
| `0x07D4` | `float` | Stamina |
| `0x08F0` | `Matrix4x4` | Entity world transform |
| `0x094C` | `CollisionMesh*` | Collision mesh |
| `0x0A88` | `CollisionMesh*` | Additional conditional collision mesh |

### Droideka runtime entity

The Droideka uses a vehicle-like runtime entity and does not expose the normal
soldier pose/statistics path. Its runtime type is identified by vtable RVA
`0x0039B0AC` relative to `BattlefrontII.exe`.

| Offset | Type | Function |
|---:|---|---|
| `0x0030` | `uint32_t` | Runtime form flags; `(value & 0x0C) != 0` identifies rolling form |
| `0x0148` | `Vec3` | Runtime world/aim position used when the standard soldier path is unavailable |
| `0x0200` | `void*` | Droideka-specific class/runtime definition; not a standard `EntityClass` |

The normal `Entity + 0x02E4`, `+0x0510`, and `+0x078C` paths can be null for
this entity type and must not be required for Droideka detection.

## `Weapon`

| Offset or RVA | Type | Function |
|---:|---|---|
| `Weapon + 0x0060` | `WeaponClass*` | Weapon-class definition |
| `0x003B057C` | vtable RVA | Common firearm weapon instance |
| `0x003B12A4` | vtable RVA | Launcher weapon instance |
| `0x003B1988` | vtable RVA | Fusion-cutter weapon instance |

## `WeaponClass`

| Offset or RVA | Type | Function |
|---:|---|---|
| `WeaponClass + 0x0030` | inline `char[]` | Internal weapon-class name |
| `0x003B0674` | vtable RVA | Common firearm class definition |
| `0x003B139C` | vtable RVA | Launcher class definition |
| `0x003B1950` | vtable RVA | Fusion-cutter class definition |

Current equipped weapon:

```text
slotIndex = *(uint8_t*)(Entity + 0x500)
weapon    = *(Weapon**)(Entity + 0x4E0 + slotIndex * 4)
class     = *(WeaponClass**)(weapon + 0x60)
name      = (char*)(class + 0x30)
```

## `Stats` — size `0xE0`

| Offset | Type | Function |
|---:|---|---|
| `0x0000` | `float` | Direction/view X component 1 |
| `0x0008` | `float` | Direction/view Z component 1 |
| `0x0020` | `float` | Direction/view Z component 2 |
| `0x0028` | `float` | Direction/view X component 2 |
| `0x0030` | `Vec3` | World position |
| `0x0054` | `float` | Current health |
| `0x0058` | `float` | Maximum health |

## `Aimer` — size `0x404`

| Offset | Type | Function |
|---:|---|---|
| `0x0048` | `Vec3` | Aim rotation |

## High-resolution skeleton

```text
Entity + 0x510 -> pose workspace
pose workspace + 0xE0 -> Matrix4x4 bone array
bone stride = 0x40
bone translation = matrix[12], matrix[13], matrix[14]
Entity + 0x8F0 -> entity world Matrix4x4
```

Model-space bone positions are transformed through `Entity + 0x8F0`.

### Human bone indices

| Function | Indices / chain |
|---|---|
| Head | `15` |
| Spine | `2 -> 3 -> 4 -> 5 -> 15` |
| Right arm | `5 -> 6 -> 7 -> 8` |
| Left arm | `5 -> 10 -> 11 -> 12` |
| First leg | `2 -> 17 -> 18 -> 19 -> 20` |
| Second leg | `2 -> 21 -> 1 -> 13 -> 16` |
| Feet | `16`, `20` |

### Battle-droid bone indices

| Function | Indices / chain |
|---|---|
| Head | `16` |
| Neck/upper chest | `15` |
| Spine | `3 -> 4 -> 5 -> 15 -> 16` |
| Right arm | `5 -> 6 -> 8 -> 9` |
| Left arm | `5 -> 10 -> 12 -> 13` |
| First leg | `3 -> 18 -> 19 -> 20 -> 21` |
| Second leg | `3 -> 22 -> 1 -> 2 -> 14` |
| Feet | `14`, `21` |

## Low-resolution skeleton

```text
Entity + 0x790 -> SoldierAnimatorLowRes*
SoldierAnimatorLowRes + 0x30 -> Matrix4x4 bone array
bone stride = 0x40
```

## `SpawnDisplay`

```text
module base + 0x1B9AE5C -> SpawnDisplay*
```

| Offset | Type | Function |
|---:|---|---|
| `0x0018` | `int32_t` | Display state: `2` closed, `3` open, `4` waiting for respawn |
| `0x1D74` | `void*` | Selected class UI context; mirrors `*(EntityClass + 0x164)` |
| `0x1D84` | `void*` | Selected/highlighted class UI object |
| `0x1E10` | `EntityClass*` | Selected class definition |
| `0x1FDC` | `Character*` | Local character used by class selection |
| `0x1FEA` | byte | Class-display lifecycle-active state |
| `0x2010` | `void*` | Team-specific class-menu context; mirrors `Team + 0x794` |
| `0x2018` | `EntityClass*` | Selected class definition |
| `0x2054` | `int32_t` | Team ID presented by the class menu |
| `0x2058` | pointer | Team-dependent display entity/context |
| `0x205C` | pointer | Team-dependent display entity/context |

## Camera structures

### `CameraManager` — size `0x104`

| Offset | Type | Function |
|---:|---|---|
| `0x0024` | `GameCamera*` | Active game camera |
| `0x0028` | `ChaseCamera*` | Chase/alternate camera |

### `GameCamera` — size `0x1D0`

| Offset | Type | Function |
|---:|---|---|
| `0x0030` | `Matrix4x4` | Camera matrix 1 |
| `0x0070` | `Matrix4x4` | Camera matrix 2 |
| `0x00B0` | `Matrix4x4` | Projection matrix; first element is horizontal projection scale |
| `0x00C4` | `float` | Vertical projection scale within the projection matrix |
| `0x00F0` | `Matrix4x4` | View-projection source matrix |
| `0x0138` | `float` | Reciprocal horizontal projection scale before the active zoom multiplier |
| `0x013C` | `float` | Reciprocal vertical projection scale before the active zoom multiplier |
| `0x0140` | `float` | Active zoom/projection multiplier (`1.0` normal, approximately `2.0` and `8.0` in the captured sniper zoom stages) |
| `0x015C` | `Matrix4x4` | Camera matrix |

### `ChaseCamera` — size `0x140`

| Offset | Type | Function |
|---:|---|---|
| `0x0010` | `Matrix4x4` | Chase-camera matrix |

## Direct3D 9 vtable indices

| Index | Function |
|---:|---|
| `16` | `IDirect3DDevice9::Reset` |
| `42` | `IDirect3DDevice9::EndScene` |

## Executable identity

The static FPS and sensitivity offsets below apply only to this analyzed
32-bit executable.

| Property | Value |
|---|---:|
| Original file size | `4,357,632` bytes |
| Canonical SHA-256 | `3BFDB0931885DE776126957AA4883B691FC391B58126FF639E36592309D75E90` |
| Preferred image base | `0x00400000` |
| `.text` RVA / file offset | `0x00001000` / `0x00000400` |
| `.text` characteristics | `0x60000020` — execute/read, not writable |
| `.data` RVA / file offset | `0x003DE000` / `0x003DC200` |
| `.data` characteristics | `0xC0000040` — read/write |

Verify the executable layout and original bytes before applying static offsets
to another build.

## FPS limit and unlock

| Purpose | RVA | File offset | Original bytes/value |
|---|---:|---:|---|
| Existing 12-byte code cave | `0x0013AB14` | `0x00139F14` | Twelve `CC` bytes |
| FPS push instruction | `0x0013AB7B` | `0x00139F7B` | `6A 50` (`push 80`) |
| FPS unlock branch | `0x001DED97` | `0x001DE197` | `74` |

The unlock patch changes the branch byte to `75`.

For FPS values from `30` through `127`:

```text
file 0x00139F14: CC CC CC CC CC CC CC CC CC CC CC CC
file 0x00139F7B: 6A <FPS as one byte>
file 0x001DE197: 75
```

For FPS values from `128` through `1000`:

```text
file 0x00139F14: 68 <FPS as little-endian int32> 74 62 CC CC CC CC CC
file 0x00139F7B: EB 97
file 0x001DE197: 75
```

`EB 97` redirects the original FPS push site into the existing cave. The cave
uses `push imm32`, allowing values that cannot be represented by the original
one-byte immediate form.

## Gameplay-only mouse sensitivity

### Mouse-update endpoint

| Purpose | RVA | File offset | Size / original value |
|---|---:|---:|---|
| Mouse-update epilogue hook | `0x002DA6C0` | `0x002D9AC0` | 5 bytes: `5F 5E C2 04 00` |
| Executable sensitivity routine cave | `0x0036A520` | `0x00369920` | 96 reserved bytes |
| Writable X remainder | `0x003F4FE0` | `0x003F31E0` | 4 bytes |
| Writable Y remainder | `0x003F4FE4` | `0x003F31E4` | 4 bytes |
| PE `.text` VirtualSize field | — | `0x00000258` | Original `0x0036950B` |

The mouse update function ends at RVA `0x002DA6C0` with:

```asm
pop edi
pop esi
ret 4
```

At this point:

```text
[esi + 0x08] = signed per-frame gameplay X mouse delta
[esi + 0x0C] = signed per-frame gameplay Y mouse delta
```

The menu cursor has already consumed the original unscaled mouse deltas before
this endpoint. Scaling these two fields therefore changes gameplay sensitivity
without changing menu-cursor movement.

### Hook and cave layout

The 5-byte epilogue is replaced by this relative near jump:

```text
E9 5B FE 08 00
```

Relative displacement:

```text
0x0036A520 - (0x002DA6C0 + 5) = 0x0008FE5B
```

When the cave is active, the `.text` VirtualSize field is expanded from
`0x0036950B` to `0x00369580`, which includes the complete routine. At sensitivity
`1.00x`, the original VirtualSize and epilogue are restored, the 96-byte cave is
cleared, and both writable remainder values are cleared.

Mutable remainder state must remain in writable `.data`. Placing it in `.text`
causes an access violation when the routine attempts to update it.

### Sensitivity representation and arithmetic

The multiplier is stored as an integer percentage from `10` through `100`,
representing `0.10x` through `1.00x`. Each axis is processed independently:

```text
accumulated  = rawDelta * percentage + previousRemainder
scaledDelta  = accumulated / 100
newRemainder = accumulated % 100
```

Signed `idiv` is required so negative mouse movement is handled correctly.
Retaining the signed X/Y remainders prevents low multipliers such as `0.10x`
from discarding repeated one-count mouse movements.

### Injected routine operation

The cave routine:

1. Preserves `EAX`, `EBX`, `ECX`, and `EDX`.
2. Uses `call`/`pop ecx` to obtain a position-independent base.
3. Loads divisor `100` into `EBX`.
4. Reads `[esi+0x08]`, multiplies by the percentage, adds the X remainder,
   performs signed division, writes the quotient back to `[esi+0x08]`, and
   stores the remainder at RVA `0x003F4FE0`.
5. Repeats the operation for `[esi+0x0C]` and RVA `0x003F4FE4`.
6. Restores the preserved registers.
7. Executes the original `pop edi; pop esi; ret 4` epilogue.
8. Ends with marker bytes `42 46 53 31` (`BFS1`).

The position-relative remainder displacements are calculated from
`SensitivityCaveRva + 9`, the address captured by the routine's `call`/`pop`.

### Patch-owned-region normalization

To validate a patched executable against the canonical SHA-256, normalize a
copy in memory as follows before hashing:

```text
file 0x001DE197             = 74
file 0x00139F14, length 12  = CC CC CC CC CC CC CC CC CC CC CC CC
file 0x00139F7B             = 6A 50
file 0x002D9AC0             = 5F 5E C2 04 00
file 0x00369920, length 96  = all zero
file 0x003F31E0, length 8   = all zero
file 0x00000258             = little-endian 0x0036950B
```

The resulting file must be exactly `4,357,632` bytes and hash to the canonical
SHA-256 listed above.
