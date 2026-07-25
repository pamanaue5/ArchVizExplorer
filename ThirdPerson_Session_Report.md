# Third-Person Walkthrough — Overnight Session Report (CP0–CP1)
*Claude (Opus 4.8), 23 Jul 2026. Read this, then press Play. Nothing in your core blueprints was changed.*

## TL;DR
- **CP0 (verify live graphs): done — graphs are clean and exactly as hoped.** Your `Switch_Pawn` system is ready to extend.
- **CP1 (prototype): done in an isolated sandbox and proven.** The Third Person character **stands on your Cesium ground** at BP_POI11. I built `BP_WalkTest`, compiled it clean, and dropped one into the level. **Press Play to try it.**
- **I stopped before touching `BP_Explorer_PC` / the enum** — that's the crash-historied core, you'll want to watch it PIE-test step by step, and it's your call how to fold it in. CP2–CP4 are planned and ready for your go-ahead.
- **One big gotcha found** (would've eaten a live session): BP_POI11 floats ~12 m above the ground; the character must be **line-traced down to the terrain**, not dropped at the POI.

---

## CP0 — what your project actually does
- `BP_Explorer_PC` has a **`Switch_Pawn(PawnType)`** custom event. It: stores the current look-yaw in `Controller_Yaw` → `Switch on PawnType_Enum` → **Possess** the pawn → restore rotation → toggle menu + POI visibility.
- Pawns are **placed in the level**, referenced through the Game Instance `BP_Explorer_GI` (`Explorer_Main_Pawn`, `Explorer_360_Pawn`, `BP_POIs`, `POI_Selected`).
- `PawnType_Enum` has **two** values: `Explorer_Main_Pawn` (orbit) and `Explorer_360_Pawn`. **We add a third** for the walkthrough.
- **This is the ideal structure** — enum-driven possession with yaw storage already exists. We inherit it, exactly as you decided.

## CP1 — the prototype + what I proved
I temporarily placed a Third Person character at BP_POI11 and photographed it (no PIE, nothing saved), then built the sandbox blueprint.

**Proven:** the UE5 mannequin loads and **stands cleanly on your Cesium terrain** at the site (see `scratchpad` images / the ones I sent in chat).

**The gotcha (important):**
- BP_POI11's pivot is at Z ≈ **-49621**, but its marker floats up at the *route/camera altitude*.
- `snap_to_ground` snapped the character to **Z ≈ -45276** — that's the POI's *own* collision, up in the air. Left like that, the character hovers ~12 m above the ground.
- The **real Cesium ground** under BP_POI11 is **Z ≈ -46618** (found by tracing straight down). Character origin sits on it at **Z ≈ -46528**.
- **➜ The real spawn logic must line-trace down from the POI and use the terrain hit** — I'll bake that into CP2 so it works for any POI, not just this one.

## The sandbox: `BP_WalkTest`  (in `/Game/_TP_Test/`)
A throwaway test actor. It does **not** touch your PC, enum, or widgets. One instance (`BP_WalkTest_Runner`) is in the level.

**How to test:** just press **Play**. After ~1.5 s it will: spawn the Third Person character at BP_POI11, possess it, add the `IMC_Default` input, blend the camera to it, and print *"TP Walkthrough ACTIVE."* Then **WASD to move, mouse to look.** Press **Stop** to end.

**What "good" looks like:** you're controlling the mannequin, standing on the ground by the route line, able to walk around the site.

**Known quirks (expected, not bugs):**
- The camera may fight slightly, because the PC's `Check_Cursor_Movement` still runs on Tick. That's exactly the **`bWalkMode` Tick-gate** from your locked-in decision #7 — it's a CP2 task, not a problem with the prototype.
- Menus from the Explorer still appear. Ignore them for this test.

**To remove when done:** delete `BP_WalkTest_Runner` from the outliner, and (optionally) the `/Game/_TP_Test` folder. Nothing else depends on it.

---

## Ready to build on your go-ahead — CP2–CP4
| CP | What | Notes |
|----|------|-------|
| **2** | Add `ThirdPerson` to `PawnType_Enum`; build the real Enter path in `BP_Explorer_PC` (line-trace-to-ground spawn + possess, off the focus-timeline's *finished* event, **not** Tick); gate `Check_Cursor_Movement` with `bWalkMode`; orbit pawn goes dormant. | I'll build it in **new function graphs** so your working EventGraph is never rewritten. You PIE-test each piece. |
| **3** | Exit path → possess orbit pawn → **refocus BP_POI11** (your point 6). | Reuses your existing Focus timeline. |
| **4** | Buttons: a **"Walk here"** + **"Exit walkthrough"** in **`BP_Mahi_Widget`** (it's currently blank — perfect canvas), surfaced via the master-menu taskbar that already references it. | Confirm Mahi is the intended panel. |
| **5** | Black_Eye camera pass (testing). | Plugin's `BlackEyeLookAt` nodes are available. |

## The only things I need from you in the morning
1. **Press Play** and tell me how the walk feels (or if anything misbehaves).
2. Confirm **`BP_Mahi_Widget` is the walkthrough panel** (my read: yes).
3. Then say the word and I'll start **CP2** — extending the enum + `Switch_Pawn`, checkpoint by checkpoint, with you watching each PIE test.

Nothing is broken; nothing in your core was touched. Ka pai te moe. — Claude
