# Third-Person Walkthrough — Prep Checklist
*For the next build session. Project: ArchVizExplorer (UE 5.8, Cesium), level TKOPPARCHVIZ. Written 22 Jul 2026.*

The **goal**: from the existing orbit UI, spawn & possess a walking character at chosen POI points, walk at human scale, then exit back to the orbit camera — wired into `BP_MasterMenu_Widget` and `BP_Mahi_Widget`.

Because the Unreal MCP is connected, **most of the actual building (enum entry, Blueprint graphs, actor placement, widget wiring) I can do programmatically next session.** So your prep is mostly **decisions** + a few things only you can judge. Nothing below needs to be built now.

---

## What I've already confirmed (you don't need to re-gather this)
- Current pawn = `BP_Explorer_Pawn`, a SpringArm orbit/zoom camera. Switching is driven by `PawnType_Enum` inside `BP_Explorer_PC`.
- `PawnType_Enum` has **2 entries** (`Explorer_Main_Pawn` + 1). **No Third-Person entry yet** — I'll add one.
- POI system exists: `BP_POI`, `POI_Info_Struct`, `POI_Type_Enum` (POI / POI_Center / POI_Filter).
- Character content in the project is **UE4-era only** (SK_Mannequin / old ThirdPersonCharacter). No UE5 Manny/Quinn, no MetaHuman yet.
- Level is `/Game/Map/TKOPPARCHVIZ`. MCP live on port 8000.

---

## Decisions I need from you (the real prep)

### 1. Which character? ⭐ biggest one
- **A — Add the UE5 Third Person feature pack** (Content Browser → **Add** → **Add Feature or Content Pack** → **Third Person**). Gives `SKM_Manny`/`SKM_Quinn` + `ABP_Manny` + Enhanced Input assets. Cleanest, recommended.
- **B — MetaHuman** (plugin already enabled). Higher fidelity, heavier, more setup.
- **C — Reuse the UE4 mannequin** already in the project. Least new content, older rig.
- **➜ Your prep:** pick one. If **A**, add the feature pack before we start. If **B**, generate/import the MetaHuman. If **C**, nothing to add.

### 2. Which POIs are "walkable", and where exactly does the character stand?
A POI's transform may sit at camera-pivot height — dropping a character there can spawn it mid-air or inside a mesh.
- **➜ Your prep:** give me a **list of the POIs** you want walkable, and for each a **ground-level spawn spot + facing direction**. Easiest way: in-editor, move the viewport to where a person should stand, and either drop a **Target Point** actor there (name it e.g. `TP_Spawn_<POIname>`) or just tell me the coordinates/POI. Even "these 3 POIs, facing the building" is enough to start.

### 3. Cesium collision — will the character have ground to stand on? ⭐ classic ArchViz trap
A walking pawn **falls through Cesium tiles** unless the tileset generates collision.
- **➜ Your prep:** on your **Cesium3DTileset** actor(s), confirm collision/physics meshes are enabled (tileset → **Collision** / *Create Physics Meshes*). If the walkable area is small, an alternative is a hidden collision floor at the site. Tell me which approach you want, or just flag "not sure" and I'll inspect it.

### 4. The two menus — what's the intended flow?
`BP_Mahi_Widget` is new since the last plan, so I need its role.
- **➜ Your prep, answer in plain words:**
  - Where does the **"Enter walkthrough / Walk here"** button live — POI info panel? Master menu? Mahi widget?
  - How do you choose **which** POI to walk to — click a POI, pick from a list, or one button that drops you at a default POI?
  - How do you **exit** back to orbit — button, Esc, or both?
  - What is `BP_Mahi_Widget` for vs `BP_MasterMenu_Widget`?

### 5. Movement & camera feel
- **➜ Your prep, confirm:** WASD + mouse-look only, or also jump / sprint / crouch? Camera = over-the-shoulder third-person, or first-person eye-height? Walk speed = normal human pace or faster?

### 6. Exit behaviour (minor — I'll default if you don't mind)
On exit, return the orbit camera to where it was before (default), or refocus on the POI you just visited?

---

## Housekeeping before the session
- **Back up the project** (zip). Given the July-2025 crash history, do this first.
- **Save all** in-editor; make sure we're **not in PIE** when we start.
- Have the editor **open on TKOPPARCHVIZ** with the MCP server running (port 8000).

## Locked-in rules I'm already carrying (no action needed — just so you know I've got them)
- Extend `PawnType_Enum` + the existing pawn-switch path; don't build a parallel system.
- Explorer pawn goes **dormant (hidden), never destroyed**.
- **No** possession swaps / Destroy / component register from **Tick** — that was the `!bPostTickComponentUpdate` crash. UI/input events only.
- Build graphs fresh (via MCP), compile & verify after each step; one checkpoint at a time.

---

### Fastest path to "yes let's go"
If you want the minimum to start: **(1)** pick the character, **(2)** enable Cesium collision or say "use a floor", **(3)** name 1 POI to walk to first. We can prototype that single walk-to-one-POI end to end, then expand. Ka pai.
