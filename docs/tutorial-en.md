# CORTAO Ambient Occlusion 1.2.1 — User Guide

[简体中文](tutorial-zh-CN.md) · [Documentation home](../README.md)

CORTAO is a screen-space ambient occlusion (AO) effect. It reads scene depth and deferred GBuffer normals, then composites visibility into scene color before the regular post-processing chain. It can add contact detail and depth to creases and corners. It does not replace light shadows, global illumination, or material AO.

## 1. Check compatibility

| Item | Requirements and validation scope |
| --- | --- |
| Platform | **Windows 64-bit (Win64)**. The plugin descriptor restricts the supported platform and its modules to Win64. |
| Engine | Use the package for your UE minor version: `Cortao_5.6` for 5.6, `Cortao_5.7` for 5.7, or `Cortao_5.8` for 5.8. The tested patch builds are **5.6.1**, **5.7.4**, and **5.8.4**, respectively. For a different patch or a custom engine build, compile and validate on that exact engine; changing the descriptor alone does not fix binary or private Renderer API compatibility. |
| Renderer | Desktop **deferred shading** with scene color, depth, and GBuffer normals. **Shader Model 5 or higher**. The view must have Lighting and Post Processing enabled. |
| Verified GPU paths | The prebuilt install packages were rendered on an RTX 4070 Ti SUPER: UE 5.6.1 on D3D11/SM5, and UE 5.7.4 and 5.8.4 on D3D12/SM6. These are specific test combinations, not a claim that every engine/RHI combination, GPU, or packaged Shipping build has been tested. |
| Unsupported views | Forward shading, mobile rendering, feature levels below SM5, stereo/VR views, scene captures (including `SceneCapture2D`), reflection captures, and planar reflection views. A dedicated server does not render CORTAO. |

Go to **Edit → Project Settings → Engine → Rendering** and search for **Forward Shading**. It must be off for this plugin. If it is on, evaluate the impact on the rest of the project before switching render paths. Under **Project Settings → Platforms → Windows**, review **Default RHI** and the targeted shader formats for that RHI. Make sure the chosen format meets the SM5-or-higher requirement and is included when you cook the game. Close and restart the editor after changing the render path or RHI; allow shader compilation to finish. Check the startup log for the RHI and shader platform actually in use when diagnosing a mismatch.

## 2. Install or upgrade

1. Close the Unreal Editor. Back up or commit the project first. Check both the project's `Plugins` directory and your engine's plugin installation for another `Cortao` copy. Keep just **one** active copy; do not merge a new package into an old one.
2. **Prebuilt install package:** Copy the complete `Cortao` directory from `Install` into `<Project>/Plugins/`. Use binaries built for the exact engine installation you are launching. Do not mix `Binaries` from one version with `Shaders` or `Source` from another.
3. **Source-only package:** Choose one of `Cortao_5.6`, `Cortao_5.7`, or `Cortao_5.8`. Copy its contents into `<Project>/Plugins/Cortao/`. It has no prebuilt binaries. Build with the C++ toolchain required by the target engine, including the appropriate MSVC toolset and Windows SDK. For a Blueprint-only project that cannot compile the plugin on open, use a matching prebuilt package, or prepare a C++ build host for that engine instead of copying DLLs from another version.
4. Check that the descriptor is at `<Project>/Plugins/Cortao/Cortao.uplugin` and that `Resources`, `Shaders`, and `Source` are alongside it. Keep the shader and source trees intact.
5. Open the project, go to **Edit → Plugins**, search for **CORTAO Ambient Occlusion**, enable it, and restart when prompted. A source install must build successfully before it can load. If Unreal reports a missing/incompatible module, check **§9** before attempting to bypass it.

For a source build outside your project, an alternative is to build the plugin with that engine's AutomationTool. Run the following from a Windows command prompt, replacing all angle-bracket placeholders. Use a dedicated output directory **outside** the plugin source folder. Use the result built for the same engine; this command does not validate rendered output or cook your game.

```bat
"<UE_ROOT>\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin -Plugin="<PLUGIN_DIR>\Cortao.uplugin" -Package="<OUTPUT_DIR>" -TargetPlatforms=Win64
```

## 3. Verify the first result

1. Open **Project Settings → Plugins → CORTAO**. Set **Enabled** on, **Quality** to **Medium**, and **Output** to **Composite**. Keep the other defaults for the first check.
2. Use a normal **Lit** perspective editor or player view. Point it at visible contact edges or corners of opaque geometry within a few meters. Empty sky, objects too far away, and views without GBuffer normals are not useful tests.
3. Set **Output** to **Before / After Comparison** and **Split Position** to `0.5`. The left shows the same scene without CORTAO; the right shows CORTAO. Other project lighting and post-processing are present on both sides. Use `0` for CORTAO across the whole view and `1` for the original across the whole view. If you're tuning from the editor quick panel, the mode is labelled **Before / after**.
4. If the composite difference is hard to see, check **Ambient Occlusion Only**. White is neutral; dark areas show reduced visibility. Return to **Composite** when finished. If the game and the editor disagree, check runtime overrides (§6–8), volumes, and console variables.

### Output modes

| Mode | What it shows |
| --- | --- |
| **Composite** | The normal scene, multiplied by CORTAO visibility. This is the mode for normal play. |
| **Ambient Occlusion Only** | Grayscale sample visibility with distance fade. It is not a preview of composite **Intensity** or **Minimum Visibility**; adjust those in Composite or Before / After. |
| **Bent Normals** | RGB-encoded view-space bent normals for diagnosis. It is neither a world-space normal view nor a material or lighting output. |
| **Before / After Comparison** | Original on the left, CORTAO composite on the right. Split Position is normalized per view and controlled by the setting; it is not a draggable viewport overlay. |

## 4. Tune and save project defaults

Outside Play In Editor (PIE), open **Window → CORTAO Quick Tuning**, in the **CORTAO** section of the Window menu. This is the editor panel, not the runtime panel. Presets load a full starting configuration, not just a single parameter: applying one re-enables AO and resets output to Composite, the divider, fade distances, and advanced values to preset settings. Choose a preset before doing fine adjustments. **Restore plugin defaults** writes the built-in defaults to the project, rather than undoing only the last change.

Edits preview live. Release a slider or commit a typed value to save it; preset and step buttons save immediately. Numeric fields support direct entry and **− / +** adjustment. **AO Strength** has a fine `0.001` step. Fade controls under **Advanced / fade & noise** use logarithmic sliders and `10 cm` steps; when the bounds cross, the other bound moves to keep the interval valid. Saved project defaults belong to the project's engine configuration (`Config/DefaultEngine.ini`); review the config diff and version it with the project. This does not save an independent runtime override or a level actor.

### Main settings

| Setting | Default / range | How to use it |
| --- | --- | --- |
| **Enabled** | On | Base setting for a World. A nearby region can blend a different enabled state. For a global off switch use `r.Cortao.Enable 0`. |
| **Quality** | Medium / Low 8, Medium 16, High 32 samples per pixel | Start with Medium; test Low for budget and High for grain. Sample count is not an FPS estimate. |
| **Radius** | `100 cm` / `1–1000 cm` | World-space sampling reach. Reduce it for localized contact detail and to avoid broad dark halos. |
| **Intensity** / **AO Strength** | `1` / `0–4` | Larger values darken the composite more. `0` makes the composite neutral; it does not suppress the diagnostic outputs. |
| **Minimum Visibility** | `0.05` / `0–1` | Raise it to protect detail in dark areas. `1` makes the composite neutral. |
| **Screen Radius** | `0.20` / `0.01–0.50` | Caps the projected reach relative to view height. Reduce it if near-camera halos are too broad. |
| **Filter** | On | Edge-aware smoothing. Turn it off temporarily when diagnosing grain or depth/normal edge bleeding. |
| **Temporal Noise** | Off | Off uses a stable sampling pattern. On changes the pattern each frame; use it with TAA/TSR and evaluate moving cameras. CORTAO has no history accumulation of its own. |
| **Fade Start / Fade End** | `5000 / 10000 cm` | Fade from full contribution to neutral over this **view-depth** interval. End must be above Start. Shorten it to limit far-field artifacts; an early fade can also hide the effect you are trying to tune. |
| **Split Position** | `0.5` / `0–1` | Only used by Before / After Comparison. It does not control the effect's strength. |

### Advanced settings

| Setting | Default / range | Guidance |
| --- | --- | --- |
| **Mip Bias** | `-4` / `-8–0` | Adjusts sampling mip selection. Leave it at default unless investigating a repeatable artifact and compare at the intended render resolution. |
| **Slope Compensation** | `4` / `1–8` | Adjusts compensation for sloped surfaces. Change independently when diagnosing sloped-surface artifacts. |
| **Backface Threshold / Boost** | `0.5 / 0` / both `0–1` | Advanced controls for backface contributions. Keep the defaults unless you have a specific reason to tune them. |
| **Filter Depth Sensitivity** | `512` / `1–2048` | Higher values restrict filtering across depth discontinuities. Applies when Filter is on. |
| **Filter Normal Sensitivity** | `128` / `0–512` | Higher values restrict filtering across changes in surface direction. Applies when Filter is on. |

**Preset selection:** **Balanced** is the general starting point. **Character Soft** and **Face and Hair Soft** use shorter/weaker occlusion to preserve detail, but affect the entire view. **Fine Detail** favors short, higher-quality AO. **Contact Shadows** strengthens contacts but does not cast directional light shadows. **Interior** and **Exterior** change reach and fade for enclosed or open spaces. **Performance** uses Low and disables the edge filter, so inspect grain and edges. **Cinematic** starts at High; profile it at the actual output resolution.

**Suggested order:** Select a preset, compare at your normal camera scale, set Radius and Screen Radius, then adjust AO Strength and Minimum Visibility. Recheck during camera motion and at multiple distances before changing fade/filter or increasing Quality. Change one group at a time. Compare with the same exposure, internal resolution/screen percentage, camera, and other lighting settings. If you already use Unreal AO or Lumen's screen-space occlusion, compare both paths independently before deciding how much visual darkening you want. CORTAO does not disable those effects automatically.

## 5. Set camera-based profiles with Region Volumes

1. In the level editor, search **Place Actors** for **CORTAO Region Volume** and place it. Move, rotate or size its box using the actor transform and **Bounds** component. It is not a collision trigger.
2. In Details, choose a **Preset**. To edit individual parameters, enable **Use Custom Options** and fill in **Custom Options**. Custom Options is a separate set, not a live overlay on the selected preset; changing the preset does not update those fields.
3. Set **Blend Distance** in centimeters (`300` by default). Weight is full inside the box, smoothly falls to zero this far **outside** it, and is zero farther away. Set `0` for a hard boundary. Camera location, not a player pawn overlap or a pixel's world position, determines weight.
4. Set **Priority** (`0` by default); higher values are applied after lower values and can override them where their coverage is high. Same-priority overlaps weight the numeric options together. Discrete choices such as Quality, Filter and Temporal Noise switch instead of gradually interpolating; avoid overlapping contradictory values when a camera crosses a boundary.
5. Move the test camera slowly through the blend band and into the box. The profile changes the **entire view**, including objects outside the box. Output mode and the comparison divider stay under the World/console controls. A disabled region blends toward no effect; conversely, an enabled region can turn AO on locally when the base World option is off. The global switch still wins.

Save the level to persist volume placement and properties. To change them at runtime, modify the volume actor in the World being rendered. Streaming an actor in or out can change which volumes affect the camera; test those transitions with your real camera path.

## 6. Control CORTAO in Blueprint

Use a Blueprint in the **World you intend to affect**. In the Level Blueprint or a local player-side Blueprint, get **Cortao World Subsystem** (via **Get World Subsystem**, class `CortaoWorldSubsystem`, or the typed subsystem getter in the node menu). Drag off the returned reference to search for the CORTAO nodes; connect it to **Target**. Run a setter from **BeginPlay** for initial values or from an explicit user/settings event for changes, not every Tick. In multiplayer, execute visual settings on the clients that render; these runtime options are not replicated settings. Separate PIE instances and separate Worlds each have their own overrides.

| Node | Inputs / result |
| --- | --- |
| **Set CORTAO Enabled** | Connect execution, subsystem Target and the **Enabled** boolean. Preserves the current quality, mode and appearance. |
| **Is CORTAO Enabled** | Pure; returns the base World enabled flag. It does not include regions, renderer eligibility or the global console switch. |
| **Set CORTAO Output Mode** | Connect execution, Target and **Output Mode**. Preserves the enabled flag and other tuning; choosing a mode does not turn AO on. |
| **Get Options** | Pure; returns the active **base** options (Blueprint, runtime debug, or project defaults), not the per-camera blended result. |
| **Set CORTAO Options** | Connect execution, Target and **Options** (`Cortao Options`). Replaces the World override with the whole supplied struct. |
| **Has CORTAO World Override** / **Has CORTAO Blueprint Override** | Pure; the first reports either a Blueprint or runtime debug override, the second only a Blueprint override. Neither reports region blending. |
| **Reset To Project Settings** | Connect execution and Target. Clears **both** Blueprint and runtime debug overrides for this World; regions and global console commands still apply. |

**Change just a few fields:** Connect `Get Options → Return Value` to `Set Members in Cortao Options → Struct Ref` (store the returned struct in a local variable first if a writable reference is required). In the node's Details, expose only the pins you want to set, such as **Radius** and **Intensity**. Wire the exec path through **Set Members** into **Set CORTAO Options** and pass the resulting struct into **Options**. Starting with **Make Cortao Options** instead may reset untouched fields to built-in values.

**Resolution order:** The base options come from **Blueprint override → runtime debug override → Project Settings** in descending priority. Camera-based volume blending is applied to that base. Global console overrides and the renderer's eligibility checks apply independently. For example, `Set CORTAO Enabled(true)` cannot force the effect in a forward view or override `r.Cortao.Enable 0`. Runtime overrides are in memory only; save your own preferences if they must survive World changes or app restarts, and reapply them for the new World.

## 7. Use the runtime quick panel

1. Start **PIE**, **Standalone**, or a Development game with a local PlayerController. Open the console and enter `Cortao.Debug`. The command toggles the panel for a local controller in the current World. Use the editor panel instead if not playing.
2. To wire your own debug input, call **Toggle CORTAO Quick Panel**, **Open CORTAO Quick Panel**, or **Close CORTAO Quick Panel** and pass the local **Player Controller** pin. There is no global F7 binding to open it. With panel focus, **Escape** or **F7** closes it.
3. Adjust values as needed. Changes are temporary per-World debug overrides; closing the panel does not reset them. **Reset to project** clears the panel's debug layer. Blueprint control locks runtime tuning and remains active after a panel reset; clear that control with the Blueprint reset node if you want the project defaults again.
4. The runtime panel switches input to **UI Only** and shows the cursor. On close it restores **Game Only** and the previous cursor visibility, not a custom input mode you had set earlier. If your game uses **Game and UI** or an existing menu stack, gate the panel to debug sessions and restore your input state explicitly after use. Do not rely on a console or debug shortcut being exposed to players in Shipping.

## 8. Console controls

| Command | Values and behavior |
| --- | --- |
| `r.Cortao.Enable` | `0` disables the effect globally; `1` allows the normal World/region rules. Default `1`. |
| `r.Cortao.Output` | `-1` follows World settings (default); `0` Composite, `1` Ambient Occlusion Only, `2` Bent Normals, `3` Before / After Comparison. This does not bypass enable checks, and comparison still uses the World's Split Position. |
| `Cortao.Debug` | Toggles the runtime panel. Requires PIE or a game World with a local PlayerController. |

Run commands in the editor/game console or enter the names alone to inspect current CVar values. Restore `r.Cortao.Enable 1` and `r.Cortao.Output -1` after diagnosis. The two CVars are process-wide; they can affect multiple PIE views in the same editor process and are not a substitute for per-World Blueprint settings. Check project config and launch arguments if the CVars revert at startup.

## 9. Package, validate, and troubleshoot

- **Before packaging:** Confirm the correct plugin is enabled in the project, the target is Win64, the intended desktop RHI and shader formats are selected, and the game map you intend to test is cooked. Apply any intended project defaults and use Composite for normal players. Remove startup diagnostics and avoid saving grayscale or split output as the shipping default.
- **Build and cook:** Use the same engine as your project. If starting with source, build the plugin for the configurations you need; Editor DLLs alone do not make a packaged game. Keep the shader files present **during cook** and let Unreal handle staging. Inspect the first error in the build/cook log rather than the final generic failure.
- **Test the output:** Start a packaged Win64 Development build first. Record its actual RHI and check the first map, camera movement, region boundaries, settings reset and input return after the debug panel. Finally test the intended Shipping build on representative hardware without editor or console assumptions. Compare an equivalent CORTAO-on/off camera view by using project or local gameplay settings. Profile at the intended internal resolution with engine GPU profiling tools where available; AO cost depends on view size, quality, filtering, hardware, and the rest of the frame.

| Symptom | What to check |
| --- | --- |
| Plugin does not appear or modules are missing | Verify the `.uplugin` is in the project `Plugins` tree and not mixed with another installation. Check Win64 and the exact engine build. For source, verify the C++ toolchain and rebuild; editing `EngineVersion` alone does not rebuild the modules. |
| No effect in any output | Check that the plugin is enabled and the editor restarted. Test a Lit main view with Lighting, Post Processing and a valid GBuffer on the deferred path. Check `r.Cortao.Enable`, World/region enable, mode and divider. Try geometry inside the fade distance and inspect AO Only. |
| Only a capture, reflection or VR view is unaffected | Verify that a normal main view works. Unsupported capture, reflection and stereo views are intentionally skipped. |
| Project Settings or the quick panel appear to do nothing | Check for a World override, a nearby region, or `r.Cortao.Output` forcing the mode. The editor panel writes defaults; Blueprint overrides win over runtime tuning. |
| Grain, halos or over-darkening | Lower Radius/Screen Radius for halos; lower Intensity or raise Minimum Visibility for darkening. Then compare filter sensitivity and Quality. Leave Temporal Noise off for diagnosis; when enabling it, check TAA/TSR and camera motion. |
| AO changes at screen edges or as geometry leaves the frame | This is a screen-space technique. Off-screen and hidden geometry are unavailable. Reduce radius or strength and check the full camera path; a high quality preset cannot restore missing scene data. |
| Emissive, translucent, skin or hair detail gets darker | CORTAO multiplies scene color, not just indirect lighting. Already-composited color can be affected and a material that does not write depth/normals need not act as an occluder. Try a softer preset or smaller radius, raise Minimum Visibility, or disable AO where needed; there is no material/object mask or selective skin exclusion. |
| Shader error or a problem only after packaging | Inspect the first CORTAO/Global Shader error and startup RHI in the log. Ensure the complete `Shaders` tree was available during build/cook, the intended shader format was cooked and the same engine and plugin version were used. Rebuild/recook after replacing mismatched files. |
| Runtime panel will not open, or input mode changes on close | Use a local PlayerController in PIE/Standalone/Game and pass it to the panel node. Console access may be unavailable in the build. On closing, restore your game's intended input mode if it is not Game Only. |
