# AQUILA — Handoff

Status as of 2026-11-12. This document covers what this project is, what was done, where everything lives, how to build and test it, and what is left.

---

## 1. What AQUILA is

AQUILA is a Rust digital audio workstation made by merging two lines of development:

| Source | Repository | Role in AQUILA |
| --- | --- | --- |
| **ML Studio** (Michael Lane / Dorfhaus) | https://github.com/Dorfhaus/ML-Studio | The base. Its interface, mixer, browser, transport, menus and file format win wherever the two differ. |
| **Lemon Studio** (ItsAlphaNeon and contributors) | https://github.com/ItsAlphaNeon/Lemon-Studio | ML Studio's ancestor. Its newer commits (through November 2026) were merged in, including multi-core mixing and new plugins. |

The merged result was renamed **AQUILA** and published to **https://github.com/ajdohaxhia/ADE-Studio**.

---

## 2. Repository and branches

**Local checkout:** this folder (the one containing this file).

**Git remotes:**

| Remote | URL | Use |
| --- | --- | --- |
| `origin` | https://github.com/Dorfhaus/ML-Studio.git | ML Studio (the fork with 48 own commits) |
| `upstream` | https://github.com/ItsAlphaNeon/Lemon-Studio.git | Lemon Studio |
| `ade` | https://github.com/ajdohaxhia/ADE-Studio.git | **AQUILA — where work is pushed** |

**Branches:**
- The local branch is `michael-lane`.
- Pushes go to both `ade/michael-lane` and `ade/main`. `main` was force-replaced once with the merge, at the owner's request; every push since has been a normal fast-forward.

**Key commits (newest first):**

| Commit | What |
| --- | --- |
| `ca8e151e` | Resolve the remaining merge test failures and add this handoff |
| `4b00b476` | Retire Lemon Studio UI tests that AQUILA's ML Studio interface replaces |
| `15e1542e` | Fix more merge regressions: undo, rack, menus, looping |
| `aa976690` | Fix merge regressions in the engine, sampler and mixer |
| `fb9f17aa` | **The merge:** Lemon Studio upstream into ML Studio, rename to AQUILA |
| `0e9cc091` | Last ML Studio commit before the merge (the fork's tip) |

- `fb9f17aa^1` is ML Studio and `fb9f17aa^2` is Lemon Studio, so `git show fb9f17aa^2:<path>` gives upstream's version of any file.
- All 48 Dorfhaus commits and Lemon Studio's history are preserved.

To publish:

```bash
git push ade HEAD:michael-lane HEAD:main
```

---

## 3. Layout of the codebase

| Path | Contents |
| --- | --- |
| `crates/os-app` | The application: windows, editors, browser, mixer, rack, transport, history (undo), menus. Binary target `lemonstudio`. |
| `crates/os-engine` | Audio engine: voices, mixing, multi-core rendering (`engine/multicore.rs`), plugin delay compensation |
| `crates/os-model` | Project model shared by everything (`MixerTrack`, `Channel`, targets in `target.rs`) |
| `crates/os-flp` | FLP / `.lmn` / `.lemon` reading and writing, legacy conversion |
| `crates/os-ui`, `os-theme` | Drawing, widgets, themes |
| `crates/os-host`, `os-clap`, `os-vst2`, `os-vst3` | Third-party plugin hosting |
| `plugins/*` | Built-in instruments and effects (sampler, ML Plasma, Tesla, EQ…) |
| `tools/bundle-app` | macOS `.app` bundler |
| `tools/logo` | Icon generation from `crates/os-app/assets/logo/aquila.png` |
| `docs/assets/aquila/` | README logo and screenshots |
| `README.md` | Rewritten for AQUILA (features, building, shortcuts, credits) |

**Naming notes:**
- `version::NAME` is `"AQUILA"`.
- Built-in plugins are mostly named "ML …"; a few keep their "Lemon …" names (Flanger, Phaser, Vocoder, Drop, the controllers).
- Mixer tracks are named "Bus N" everywhere.
- **Project files:** AQUILA saves `.lmn` (`project_files::EXT`) and opens `.lmn`, `.lemon` and `.flp`.

---

## 4. Building and testing (this machine)

### Why there is a special setup
1. **Paths with spaces break the build.** The original checkout lived under `~/Library/Application Support/...`; the active checkout is now `/Users/ade/Downloads/AQUILA/ML-Studio`. The bundled LAME (autotools/make) and ML Plasma native builds fail on paths with spaces.
2. **ML Plasma's build needs 20 GiB free** before it will compile.
3. **The selected Xcode is on an external drive whose path has spaces.**

### The setup in use
- An external-SSD **sparse disk image mounted at `/Volumes/aquilabuild`** (about 28 GB, no spaces in the path).
- **`/Volumes/aquilabuild/chk.sh`** is the helper used for every build and test. It:
  1. rsyncs `/Users/ade/Downloads/AQUILA/ML-Studio/` (corrected from the previous scratch checkout during continuation) to `/Volumes/aquilabuild/tree` (excluding `.git` and `target`), and
  2. runs cargo there with:
     ```
     DEVELOPER_DIR=/Library/Developer/CommandLineTools PYTHON=python3 CARGO_INCREMENTAL=0
     LEMON_STORAGE_CONFIG=/Volumes/aquilabuild/work/storage.json CARGO_TARGET_DIR=/Volumes/aquilabuild/target
     ```
- **`/Volumes/aquilabuild/work/storage.json`** = `{"minimum_free_gib": 6, "maximum_cache_gib": 0}` lowers ML Plasma's free-space minimum. This is for local runs only and must not go into release builds.
- **`CARGO_INCREMENTAL=0`** was added after the incremental cache (15 GB) filled the image. If space runs out, delete `/Volumes/aquilabuild/target/debug/incremental`.

Examples:

```bash
/Volumes/aquilabuild/chk.sh test -p os-app --bins -- mixer::
```

```bash
/Volumes/aquilabuild/chk.sh test --workspace --lib --bins --tests --no-fail-fast
```

**Gotcha:** the os-app test binary is `--bins` (named `lemonstudio`), not `--bin os-app`.

### Test-comparison helpers (in `/Volumes/aquilabuild/work/`)

| File | Purpose |
| --- | --- |
| `base-sorted.txt` | **Baseline:** the 110 tests that already failed in ML Studio before the merge |
| `newfail.sh <cargo test args>` | Runs tests, prints pass/fail totals and **only failures not in the baseline** |
| `ignore.py "reason" mod::tests::name ...` | Adds `#[ignore = "reason"]` to the named tests (run from the repo root) |
| `last.log` | Output of the most recent `newfail.sh` run |
| `up-mixer.rs`, `fork-mixer.rs` | Upstream / fork copies of `mixer.rs` used during the merge |

---

## 5. What was done

### 5.1 The merge (`fb9f17aa`)
- Lemon Studio's new commits were merged into ML Studio, and conflicts were resolved in favour of ML Studio's interface and behaviour.
- Duplicate definitions from both sides were reconciled (`Sprite`, `MHit::Input`, `Separator`, `DETACHABLE`, `snap_edge`…).
- Upstream's transport and browser were adapted to ML Studio's with shims.
- Rebranded to AQUILA:
  - names and packaging
  - the icon set (`.png`, `.ico`, `.icns`, nightly variants)
  - the splash `ml-splash.png`, generated from `crates/os-app/assets/logo/aquila.png`
- README rewritten with screenshots.

### 5.2 Regression fixes after the merge

**Engine and sampler** (`aa976690`)
- Restored ML Studio's Playlist audio-voice reconciliation when editing during playback. Upstream's block now skips audio-clip voices (`clip_origin`).
- Instruments' extra outputs route to their mixer tracks (`InstOut::aux`, `instruments_out`).
- Lemon Sampler: the `on` switch keeps its live value; per-note drive is implemented (`Voice::drive`, `note_drive`).
- Result: all 426 engine tests pass, and 80/80 sampler tests pass.

**Mixer** (`aa976690`)
- `slot_cpu` was 10 entries long while slots are 64, which crashed on slots past 9. Fixed in `os-model`, `os-flp` and the engine's `slot_smart`.
- Track moves remapped automation twice (ML Studio's `retarget` plus upstream's `follow_places`), which swapped targets back. Only the upstream pass and ML Studio's peak-link remap remain.
- Restored upstream track-menu commands inside ML Studio's menu: rename, random/gradient colour, icons, audio-track assign, routing, select, groups, move, insert one/multiple/from state, File ▸ states, audio editor.
- Help pages for "ML …" plugins (`help_links::plugin_page`).

**Undo, rack, menus** (`15e1542e`)
- **Plugin-window gestures:** ML Studio's per-value undo and Lemon Studio's per-gesture undo both ran. The per-gesture one is kept (`params.rs`).
- **Duplicate steps removed:** mixer colour and track moves recorded twice. Single moves are now named "Move mixer track".
- **Channel delete via the D key** now goes through `rack::delete_step`, so it is undoable.
- **`State::rows_moved` is consumed again** in `plugin_tick`. Undo/redo brings removed instruments back, including instrument-stack children.
- **Remote-control links follow moved channels** (`rack::reorder` calls `remote::follow`).
- **Advanced looping:** loop points play, save and burn (`row_loops`, `write_loops`).
- **"Color selected ▸ All"** works (`patterns::apply_swatch`).
- **Rack menus:** the channel menu gained Change icon, waveform, Wrap in container and Assign to new instrument track. The options menu gained "Selected channels ▸" and "Options ▸" submenus, keeping ML Studio's compact top level of 10 entries or fewer.
- **Playlist options:** Quantize song jumps and the performance-mode entries are placed again. The "Reset empty tracks" label is fixed.
- **Collaborate** got a real `MenuCmd`. The stray "MIDI chords" placeholder is removed.
- **Other:**
  - Saved-by text is "AQUILA x.y.z".
  - Vocoder plugin category.
  - `LINKED_EFFECTS` order matches the built-ins.
  - "Bus N" naming in the rack and mixer views.

**Tests adapted rather than code changed** (where ML Studio's design deliberately differs):
- the insert-multiple form
- the reset confirmation
- the typing keyboard being off by default
- the TRACK box doing free-drag routing
- the categorised plugin menus
- the in-app Change Log instead of "What's new"

### 5.3 Retired tests
About 50 Lemon Studio tests check UI that AQUILA replaces with ML Studio's. They are marked `#[ignore = "<reason>"]`, so they are easy to find and re-enable.

| Reason | Tests |
| --- | --- |
| Group fader semantics | 7 tests in `history.rs` |
| Animations | ripples, fades, captions, popup timing |
| Browser and drag tag | drag-tag tests, browser rows, channel-states browse, init browser |
| Windows | `win_clamp`, detach tests, detached editor layout, designer fonts |
| Plugin picker and palette | picker previews, palette, plugin manager |
| Rack view layout | 2 tests in `rack.rs` |
| Containers | 2 tests: ML built-ins saved as CLAP references need installed bundles; ML Studio's browser rows |
| ML Studio test replaced | `params::forum_editor_gesture_undo_redo`, covered by `control::tests::edits_in_a_plugin_s_window_are_undo_steps` |

To list them all:

```bash
grep -rn '#\[ignore = ' crates plugins tools
```

---

## 6. Test status before continuation

**Full workspace** (`--workspace --lib --bins --tests`):
- 2941 passed, 129 failed, 119 ignored.
- Of the 129 failures, **104 were already failing in ML Studio** (the baseline) and **25 are new** since the merge, down from 134.

### 6.1 The 25 remaining new failures

**Recording (likely a real bug, investigate first)**
- `smart_recording::tests::smart_mode_arms_without_a_question_and_writes_automation_clips`
- `record::tests::records_mixer_controls_the_tempo_and_plugin_parameters`
- `record::tests::takes_the_clip_just_before_in_song_mode`
- `record::tests::option_buttons_menus_and_keys` (expects Count-in length / Pre-roll entries)
- `track_rec::tests::an_armed_track_records_into_the_playlist`

**What's known about the recording failures:**
- `record.rs`, `smart_recording.rs`, `mixer_recording.rs`, `track_rec.rs` and `prefs.rs` are **identical to ML Studio's**, and so is the `Cmd::Record` handler in `playlist.rs`.
- Test preferences are thread-local (`settings_file.rs`), so this is not leakage between tests.
- `Cmd::Record(Opt::Automation)` toggles automation **off** (it defaults to on), yet ML Studio's test expects it on after `toggle_record`. In ML Studio's baseline this test passed, so something reached through `App::new` or `toggle_record` in merged code (app.rs or the transport) must behave differently.
- `records_mixer_…` also shows a recording making two undo steps: "Recorded items" plus "Create automation clip" from smart recording.

**App (os-app)**
- `app::tests::tempo_and_time_settings_are_undo_steps`: dragging the tempo display doesn't change the tempo in ML Studio's transport layout (`tempo_digits` hit area). This could be a UI difference or a bug.
- `app::tests::alt_click_resets_controls_without_dragging`: Alt+click gives 0.8, expected 0.23 (reset value).
- `app::tests::selection_fixes_mixer_reset_and_group_solo`: one value 0.8 vs 0.95.
- `app::tests::piano_ctrl_drag_selects_in_draw`: Ctrl+drag in Draw mode.
- `app::tests::main_menu_titles_switch_on_first_press`: expects an enabled "Undo history" entry.
- `app::tests::settings_tabs_can_be_hidden_and_restored_from_visibility`: an extra tab index (2) in the tab order.
- `app::tests::channel_rack_window`, `app::tests::rack_preview_mute_button_and_preview_under_the_steps`: rack window layout; probably UI differences.
- `init::tests::create_automation_clip_lists_the_control_while_the_check_is_on`: the "Automation" rack group or filter.
- `playlist::tests::renaming_a_channel_clip_renames_its_channel`: the rename box targets `AudioClip(0)` instead of `Rack(Channel(1))`.
- `playlist::tests::slot_window_menu_entries`: a plugin-window menu entry (`Plugin(Picker(Fx(Some(2))))`).
- `remote::tests::view_remote_control_lists_the_links`: the VIEW menu's Remote control entry.

**Other crates**
- `editor::tests::drum_loop_stretch_and_noise_gating`, `editor::tests::time_stretch_and_pitch_shift` (`plugins/tesla`)
- `editor::tests::tab_wheel_moves_a_step_per_notch_and_adds_up_fractions`, `editor::tests::the_knob_captions_stay_clear_of_band_1s_knobs` (`plugins/lemon-parametric-eq-2`)
- `editor::tests::the_newer_pages_wordmarks_end_6_px_from_the_right_edge` (`plugins/lemon-small-fx`; the wordmark is still "Lemon Filter")
- `project::legacy::tests::an_old_file_opens_and_saves_converted` (`crates/os-flp`): legacy file conversion; worth checking, because it affects opening old projects.
- `tests::a_nightly_is_an_app_of_its_own` (`tools/bundle-app`): Info.plist / `CFBundleIconName` after the rename.
- `tests::paints_text_and_shapes` (`crates/os-ui`): text width assertion (`w > 40.0`).

### 6.2 Suggested order
1. Recording (section 6.1): the most likely user-facing bug.
2. `project::legacy::...` (opening old projects).
3. `tempo_and_time_settings_are_undo_steps`, `alt_click_resets...` and `renaming_a_channel_clip...`, which may be real.
4. Bundle-app nightly test (packaging).
5. The rest: decide case by case whether to fix the code, adapt the test to ML Studio's design, or retire it with `ignore.py` and a reason.

---

## 7. Merge patterns to watch for

These caused most of the regressions, so check for them when touching merged code:

- **Double handling:** ML Studio and Lemon Studio each added their own mechanism for the same thing (undo recording, target remapping, gesture handling). Look for a fork-side `history::record` or remap inside code that upstream already wraps with `control::edit_app` or `follow_places`.
- **Lost glue:** functions that exist but are never called after the merge, such as `rack::moved_link` and the consumer of `State::rows_moved`. A quick check is `grep` for definitions with no callers.
- **Menus:** the merge usually kept ML Studio's menu builders and dropped upstream entries. Compare with `git show fb9f17aa^2:<file>`.
- **Array sizes:** ML Studio has 64 mixer slots; some upstream code assumed 10.
- **Renames:** tests and lookups still using "Lemon X" names, `.lemon` extensions, or "Insert N" instead of "Bus N".

---

## 8. Open decisions for the owner

- Whether to keep or re-enable any retired Lemon Studio UI tests, which would mean porting those UI features into ML Studio's interface.
- Whether the "Bus N" naming (now everywhere) is wanted, or ML Studio's earlier "Insert N" in the rack and mixer views.


## 9. Continuation on 2026-11-12

This section supersedes the remaining-work list in section 6. These changes were committed and pushed as `ca8e151e`.

### Fixes and test updates

- Corrected `/Volumes/aquilabuild/chk.sh` to sync the active Downloads checkout instead of the previous scratch checkout.
- Recording: audio clips honor their explicit/inherited mixer route; smart capture no longer also writes pattern controls. Remote faders, plugin parameters and container levels use smart capture, including Start on input. Clip extension preserves ML Studio's gap behavior but avoids occupied rows, including assigned instrument lanes.
- Grouped mixer fader resets preserve every changed fader in one undo step; resetting default values adds no step.
- Restored EDIT → Undo history and Current project → Remote control. Project data remains immediately before Samples.
- Legacy imports that are unchanged after conversion preserve the event stream rather than append default-only native chunks.
- Nightly packaging uses AQUILA Nightly as the display name and has one microphone permission key; its test checks the actual ICNS icon setup.
- Restored Tesla's right-click Drum Stretch action and fractional wheel stepping over the modern EQ type selector. Fixed effect wordmark right alignment.
- Adapted tests for intentional ML Studio behavior: tempo drag speed/scaling, individual audio-clip names, instrument palette, rack preview/LED arrangement, additive piano selection, Recording settings tab, categorized effect menus, template rack groups, recording filters/gap semantics, and modern plugin geometry. No new ignored tests were added.
- Made Chop host-audio tests initialize at their explicit 48 kHz render rate instead of inheriting the machine's 44.1 kHz device rate.
- New integration tests cover smart plugin automation without duplicate pattern controls and remote fader/container recording, including Start on input and disabled filters.

### Verification

The first full workspace run reported 2964 passed, 107 failed, 119 ignored. All original 25 listed merge failures were resolved. The three additional nonbaseline failures were a browser section ordering assertion (corrected) and two Chop host tests being investigated for a device-rate/fixture-rate mismatch. Final verification is recorded below.


Final full workspace command:

```bash
/Volumes/aquilabuild/work/newfail.sh --workspace --lib --bins --tests --no-fail-fast
```

**2968 passed, 104 failed, 119 ignored. Zero failures outside the pre-merge baseline.** All 25 original remaining merge failures are resolved. The final run includes the new remote/container/plugin recording tests, assigned-lane collision coverage, and explicit-rate Chop tests. No new ignores were added. The 104 pre-existing failures remain; coverage percentage was not measured. `git diff --check` passes. Code review found no remaining actionable issues after the recording follow-ups.

Final log: `/Volumes/aquilabuild/work/last.log`. First continuation full run: `/Volumes/aquilabuild/work/continuation-first.log`. Chop rate diagnostic: `/Volumes/aquilabuild/work/chop-red.log`.

Next work, if requested: investigate the 104 baseline failures; do not classify them as unresolved merge regressions. The continuation's changes and this handoff are committed and pushed (`ca8e151e`).

---

## 10. Session paused (2026-11-12, evening): start here next time

### State
- Pushed: `ca8e151e` on `ade/main` and `ade/michael-lane`. Every merge regression is resolved; the full workspace had 2968 passed and 104 failed, and all 104 failures were already failing in ML Studio before the merge.
- **Uncommitted local work** (below) started on those 104 pre-existing failures. **None of it has been committed or pushed.**
- The build helper `/Volumes/aquilabuild/chk.sh` and `/Volumes/aquilabuild/work/newfail.sh` point at `/Users/ade/Downloads/AQUILA/ML-Studio` (fixed this session). The disk image `/Volumes/aquilabuild` must be mounted.

### Uncommitted changes, by file
| File | Change | Verified? |
| --- | --- | --- |
| `HANDOFF.md` | Out-of-date lines fixed (commit table, "not committed" notes); this section | n/a |
| `crates/os-app/src/vocal/mod.rs` | Test fixture: each test gets its own temp folder (parallel tests were rewriting one shared `Vocal.wav`, giving "Original recording changed during analysis") | Yes: all 14 `vocal::` tests pass |
| `crates/os-app/src/mpe.rs` | `mpe::chop` (Quick chop, Ctrl+U) now records an undo step (`Some("Piano roll edit")`); before, it couldn't be undone | Yes: `piano_keys::legato_chop_and_paste` passes |
| `crates/os-app/assets/native-generators-groups.json`, `native-effects-groups.json` | Factory plugin folders use the current "ML …" names (old names kept), so built-ins no longer land in "Unsorted" | Yes: all `plugin_menu::` tests pass |
| `crates/os-app/src/strip_silence.rs`, `comping.rs` | Tests search submenus (ML Studio's clip menu nests Strip silence and "Use selected time range in comp") | Yes: these 2 pass |
| `crates/os-app/src/rack.rs` (`play_project`) | While a take is soloed, its comp (parent) track is disabled in the playback copy only | **No: the test run was stopped before it finished.** Run `comping::` and `rack::` first. |

### First thing next session
```bash
cd /Users/ade/Downloads/AQUILA/ML-Studio
```

```bash
/Volumes/aquilabuild/chk.sh test -p os-app --bins -- comping:: rack::
```

```bash
/Volumes/aquilabuild/work/newfail.sh --workspace --lib --bins --tests --no-fail-fast
```

Then check that the pass count went up and that no failures outside the baseline appear, and commit and push:

```bash
git push ade HEAD:michael-lane HEAD:main
```

### The 104 pre-existing failures, triaged
Full list with messages: `/Volumes/aquilabuild/work/base-msgs.txt`. About 13 are fixed by the work above (vocal ×8, legato/chop, plugin_menu ×2, strip_silence, comping ×1, plus comping solo once verified).

**Worth checking (may be real bugs):**
- `app::tests::sampler_settings_save_and_reopen`: on save, `app.rs` (~line 2346) deliberately keeps the rack's "Follow global pitch" over the sampler's own "Affected by the main pitch". A change made only in the sampler UI may be lost. ML Studio did this on purpose; the right fix is to push host toggles into the plugin, then drop the override. Needs a decision.
- `osc3::tests::replace_puts_another_generator_on_a_channel`, `app::tests::instrument_template_routes_and_live_reroute`, `app::tests::pattern_click_and_channel_insert_anchor` (unwrap on None)
- `app::tests::add_audio_and_automation_clip_channels` (rack filter becomes Unsorted, expected All)
- `app::tests::piano_tools` (the Delete tool swipe removes nothing)
- `wav::tests::writes_edisons_chunk_set_in_order` (`plugins/tesla`): LIST INFO size 22 vs 18, probably the longer "AQUILA" software name
- `plugins/lemon-small-fx` `factory_presets_list_every_parameter_and_hold_fls_states`: a preset file is missing

**Design conflicts (ask the owner):** tool keys ("p" is Draw in the piano roll; the playlist test expects Paint), typing keyboard off by default (`step_edit` ×5), "Undo history" now in the EDIT menu (3 `history` tests expect the old menu), "Soft takeover" vs "Pickup" wording.

**Environment or stale (likely retire or adapt):** `ai_control` (local network), `tools/notices` (needs network), `tools/watermark` (header check), `splash`, theme, toolbar and window-layout pixel tests, `win_clamp` rounding, browser tests that assume Windows paths (`E:\Pro Samples`).

### Update (same evening, after resuming)
- The take-solo change in `rack.rs` is **verified**: 55/55 `comping::` and `rack::` tests pass.
- Also fixed:
  - `osc3` replace test (searches the categorised plugin menu)
  - `instrument_template_routes_and_live_reroute` (falls back to `HOME` and skips when the local file is missing)
  - retired `pattern_click_and_channel_insert_anchor` (Lemon Studio's two-column channel menu)
- **Full workspace:** 2983 passed, 88 failed, 120 ignored. Down from 104 pre-merge failures; none are new.
- **Flaky (not a regression):** `plugins/lemon-sampler` `params::tests::editor_edits_reach_the_sound_without_the_host` failed once in the full parallel run and passes alone (80/80, three runs). Sampler live values sit in a table shared by all tests, so parallel tests can collide. Fix by isolating the table per test or serialising these tests.
- **The two suspected bugs were checked (later the same evening):**
  - Rack filter after undo: undo restored the filter correctly. The app starts on Unsorted where the test assumed All, so the test now sets All. Real fix kept: `history::put_rack` marks the restored selection as seen, so Auto switch display filter can't move the restored filter again.
  - Piano roll Delete tool: works. In AQUILA, D duplicates notes by default (ML Studio's "D duplicates notes" option), and Ctrl+A keeps a manual time range. The test was adapted.
- **Full workspace now:** 2986 passed, 85 failed (all pre-merge), 120 ignored. Remaining list: `/Volumes/aquilabuild/work/last.log`; triage notes earlier in this section.
- **Reminder:** the build image lives at `/Volumes/SanDisk EXTREME SSD/aquila-build/aquilabuild.sparseimage`. Mount it with `hdiutil attach <that path>` (mounts at `/Volumes/aquilabuild`) and detach it before unplugging the SSD.

---

## 11. ML Studio v2026.11.12.1 and v2026.11.13.1 merged (2026-11-13)

**Owner's decision:** AQUILA now uses **ML Studio's audio engine** for instrument and mixer scheduling, not Lemon Studio's `multicore.rs`.

### What changed
- **Merged `origin/michael-lane` (`b7b0748a`)**: about 205 files, including ML Studio's vendored `cpal`, attached mixer / instrument tracks, the audio editor (`audio_edits`), the voice assistant, export advanced mode, latency presets, device extra outputs, worker limit and priority, smart disable, and its new parameter host events (`ParamEvent::Sync`).
- **Engine:** ML Studio's `instrument_workers`, `mix_workers` and `mixer_graph`, with its `render_instruments`, `mix_serial` and `mix_graph`. **`multicore.rs` was deleted**, along with `mix_track`, `instrument_job`, `route_out`, `LoadGate` and `Engine::set_threads`.
- **Kept from Lemon Studio** (ML Studio never touched these files):
  - the vocoder, filter, flanger and phaser effects; peak and formula controllers; formula; chain
  - the `auto_bypass` module and `rec.rs`
  - `run_slot`'s slot auto-bypass: ML Studio's callers now store its result in `fx.skipped`, and both callers pass the tick and the switch
  - pattern control points in `param_auto` (on ML Studio's `HashMap`)
  - the `inst_later` scheduled notes and the MPE MIDI channel on live notes (added back into ML Studio's `render_instruments`)
- **No longer active, by design** (their tests are retired with `#[ignore]` reasons):
  - Lemon Studio's per-track input monitoring, direct outputs and disk recording in the engine (AQUILA already used ML Studio's transport for these)
  - channel-level auto-bypass for hosted instruments
  - unfaded hosted instruments during pause
  - per-frame automated level ramps for hosted instruments
- **Devices:** ML Studio's `Source` trait, `Mix` and extra device channels. Lemon Studio's variable-width multi-output (`set_width`, `play(out, width)`) is gone. `transport::raise_priority` now forwards to ML Studio's `set_feeder_priority(2)`. Linux real-time priority for the callback is kept.
- **`params.rs` deliberately kept as AQUILA's (Lemon Studio's parameter lists and gesture undo).** It now handles `ParamEvent::Sync` as a host baseline update. **Not yet ported:** ML Studio's non-blocking batched parameter polling (`poll_index`, `param_values_job`, `POLL_BATCH`), which avoids audio stalls with large VST3 banks, plus its two tests. That's a worthwhile follow-up.
- **Prefs:** ML Studio's `SMART_DISABLE` (key `smart_disable_v1`, plugins sleep when idle) and AQUILA's renamed `AUTO_BYPASS` (key `smart_disable`, "Auto-bypass when silent") are separate. Both appear on Audio ▸ CPU, next to ML Studio's threaded generator/mixer switches, worker limit and priority.
- **Model:** `PlaylistTrack` has both ML Studio's `attached_mixer` and Lemon Studio's `TrackLink` (similar purpose; consider unifying later).
- **Packaging:** `github-release.json` and the NSIS version are 2026.11.13.1. The release JSON's download URLs still point at ML Studio's server (michaellanemusic.de), as they did before; they need AQUILA's own release URLs before publishing.

### Not verified
- **Windows-only code** (`device/asio.rs`, `plugin_window/win32.rs`) was resolved to ML Studio's side but **cannot be compiled on this Mac**. Build on Windows before a Windows release.

### Test status after the merge
**3058 passed, 103 failed, 133 ignored.** 21 failures are outside the old pre-merge baseline:
- **9 need ML Studio's signed native vocal engine on the machine** (the tests deliberately fail without it): `audio_edits::tests::*` (7), `warp::tests::legacy_tuned_warp_recovers_original_controls`, `zipped::tests::audio_edits_portable_pending_revision_keeps_history_and_source`. Not a merge regression; install the vocal engine to check them.
- **12 to look at next:**
  - `app::tests::audio_router_numbers_drag_both_directions` (no "Browser preview track" number on the page)
  - `app::tests::global_snap` (toolbar layout)
  - `app::tests::hotfix_scrolled_settings_keep_about_clickable`
  - `app::tests::playback_repaint_matches_full`, `app::tests::tempo_tapper_flash_repaints_the_pad` (repaint regions)
  - `app::tests::playlist_colorful_waveforms`
  - `attached_mixer::tests::attached_existing_instrument_records_midi_in_its_projected_lane`: ML Studio's attached-mixer recording against AQUILA's recording path. **Likely the most important.**
  - `audio_cfg::tests::asio_buffer_is_always_driver_owned`
  - `browser_preview::tests::browser_preview_options_and_playlist_menu`
  - `channel_bounce::tests::rack_features_compact_controls_and_filtered_piano_layout`
  - `help::tests::guide_menu_search_read_back_and_visibility` (a guide topic lacks category, keywords or paragraphs)
  - `mixer::tests::mixer_batch_insert_validates_and_restores_routing_and_selection` (AQUILA's track menu always shows "Insert multiple...")

### Update after fixing the merge's integration failures (2026-11-13)
- **Fixed:**
  - Recording into an attached instrument lane: the lane's own instrument clips no longer push the take to another row (`record.rs`).
  - Full-frame repaints on every playback tick: `win_clamp::keep_all` compares rounded positions.
  - The colour change was one step too many: plain `MCmd::Color` records through `set_track_color` only.
  - Mixer-track rename step name; sidechain source names ("Bus N").
  - "Insert multiple..." follows ML Studio's visibility switch.
  - About page name; a help guide topic with one paragraph; "AI assistant settings" in the label-dispatched menu list; VST3 test tolerates `Sync`.
  - Slot auto-bypass is recorded again (`fx.skipped` from `run_slot` in `mix_serial` and `mix_workers`).
- **Full workspace: 3053 passed, 100 failed, 133 ignored. 15 failures are outside the old baseline, and none are caused by the merge:**
  - **9 need ML Studio's signed native vocal engine installed** (`audio_edits::*`, `warp::legacy_tuned_warp_recovers_original_controls`, `zipped::audio_edits_portable_pending_revision_keeps_history_and_source`).
  - **6 fail in ML Studio's own v2026.11.13.1 as well** (checked by building a clean copy of `origin/michael-lane`): `app::tests::audio_router_numbers_drag_both_directions`, `app::tests::global_snap`, `app::tests::playlist_colorful_waveforms`, `audio_cfg::tests::asio_buffer_is_always_driver_owned`, `browser_preview::tests::browser_preview_options_and_playlist_menu`, `channel_bounce::tests::rack_features_compact_controls_and_filtered_piano_layout`.
- **How to check a test against pure ML Studio:**
  1. `git worktree add --detach /Volumes/aquilabuild/mltree origin/michael-lane`
  2. `rsync -a --delete --exclude .git --exclude target /Volumes/aquilabuild/mltree/ /Volumes/aquilabuild/tree/`. ML Plasma's CMake cache needs the `/Volumes/aquilabuild/tree` path.
  3. Run cargo there with the same environment as `chk.sh`.
  4. Remove the worktree afterwards. The next `chk.sh` run syncs AQUILA back.

## 12. Lemon Studio's 41 new commits merged (2026-11-13)

ML Studio had nothing new. Lemon Studio `main` had moved from `415eaf26` to `011a03f2` (41 commits: playlist bars
following the time signature, stacked edges on overlapping notes and clips, FL-style gestures, Make unique for audio
clips, Consolidate / Export tracks rendering hosted plugins, per-pattern rack loops, undo fixes, the equivalence
harness in `crates/os-app/src/equiv.rs`). 58 conflicts in 10 files.

### Choices made

- **Rack loops per pattern:** Lemon Studio's mechanism (`rack.loops_of`, `follow_pattern`, `load_loops`,
  `store_loops`) replaces ML Studio's `read_loops` / `write_loops`. `play_project` keeps ML Studio's expansion of every
  pattern's loops, with `loops_of` as the pattern the rack holds. `pattern_seen` is still used by `pattern_tick` for
  auto-collapse and channel selection only.
- **Playlist ruler and grid lines:** drawn from `pl::Bars` (Lemon Studio), keeping ML Studio's snap-sized grid lines
  and vintage ruler digits.
- **Playlist Select tool:** ML Studio's stays. Lemon Studio's Shift-toggle box and right-click-clears-selection were
  not taken, so `PlDrag::Marquee` has no `toggle` field. The test `playlist_tool_wheel_and_select_box` became
  `playlist_tool_wheel`, written against ML Studio's toolbar order.
- **Playlist solo:** `solo_in_effect` now compares against `solo_keep` (ML Studio's folder / group aware set), so a
  solo that no longer holds ends without putting old states back.
- **Clip painting:** ML Studio's translucent bodies and stroked outlines stay; Lemon Studio's `see` closure for
  automation clips is unused.
- **Exports:** `App::start_export_of(job, p, song)` (Lemon Studio's name) forwards to ML Studio's
  `start_project_export(p, job, song)`.
- **Equivalence harness:** `Transport::sent_for_tests` returns `None`. ML Studio's transport hands updates to the
  audio thread through a slot it empties and keeps no copy, so the harness's "engine holds the project" check is off.
- **Undo of a pattern edit** no longer switches the pattern shown (Lemon Studio's change).
- `Snap::Knob` holds `Option<f64>`; `history::knob_value` returns `Option<f64>`.

### Test status

`newfail.sh --workspace --lib --bins --tests --no-fail-fast` before the last three fixes: 3076 passed, 103 failed,
140 ignored, with 3 new failures, all in os-app. After fixing them, `newfail.sh -p os-app --bins`: 1466 passed, 90
failed, 107 ignored, and the failures outside the baseline are the same 15 as in section 11. The full workspace was
not rerun after those three fixes.

Use `--lib --bins --tests`: a bare `--workspace` also builds `plugins/lemon-wave/examples/verify_host.rs`, which does
not compile.

### Still open

- Windows-only code is not compiled.
- `packaging/github-release.json` still links ML Studio's binaries; there are no AQUILA builds to link.
- ML Studio's batched parameter polling (`poll_index`, `POLL_BATCH`) is not ported: AQUILA does not poll values
  every tick, it reads them when a list is built and when a re-read is due.

### Update (same day): full run and lock-free value reads

- Full workspace after the fixes above: 3079 passed, 100 failed, 140 ignored. The failures outside the baseline are
  the same 15 as in section 11.
- `params.rs` now uses `Plugin::param_values_job()` (from ML Studio's host) for those two reads when a plugin offers
  it: the snapshot is collected in `snapshots` and run after the instance is unlocked, so a large bank does not hold
  the audio thread. Plugins without it are read under the lock as before. The os-app tests are unchanged by this
  (1466 passed, 90 failed); no test exercises the snapshot path with a real VST3.
- The build image had been unmounted; remount with
  `hdiutil attach "/Volumes/SanDisk EXTREME SSD/aquila-build/aquilabuild.sparseimage"`.

## 13. ML Studio v2026.11.13.2 merged (2026-11-13)

ML Studio published `6be4ba4c` ("Publish complete DAW v2026.11.13.2"): the product is renamed from "Lemon Studio" to
"ML Studio" throughout (stock plugins too: "ML Limiter", "ML Sampler", ...), the sampler gains arpeggiator and echo
note effects in the engine (`engine/sampler_notes.rs`), preset loads build a new plugin instance, the user guide is
rewritten with its own detachable window, and saved custom window layouts arrive (`workspace_layout.rs`). 110
conflicts in 40 files; 52 were only the rename and were resolved by script (ours where ML Studio only renamed, theirs
with "ML Studio" turned into "AQUILA" where only AQUILA had renamed).

### Choices made

- **Names:** the app stays AQUILA (`version::NAME`, `os_paths::APP_NAME`, window titles, updater messages, the
  templates' factory category). The stock plugins take ML Studio's new "ML ..." names; the effects only AQUILA has
  keep theirs (Lemon Filter, Flanger, Phaser, Vocoder, Peak Controller, Formula Controller, Drop). The user guide text
  still says "ML Studio".
- **Engine:** ML Studio's sampler note effects are in, with AQUILA's extra fields kept on `Trigger`, `notes::Event`
  and `NoteLevels` (`clip_origin`, `at`, `stop`, `colour`, `swing`, `end`, `channel`). A channel that is off still
  starts no notes (the check is at the top of `start_plain_trigger`). `run_slot` smooths, then reports the skipped
  slot.
- **Lemon Sampler DSP:** Lemon Studio's restructured voice code stays; ML Studio's `echo_pitch` and `note_off_id`
  are added to it.
- **Playhead follow (`App::auto_scroll`):** rewritten once from both: ML Studio's `visual_tick` and its "not while
  dragging in the playlist" rule, Lemon Studio's piano roll following a pattern clip in song mode.
- **VIEW > Arrange windows:** Lemon Studio's layouts first, then ML Studio's saved custom layouts; the entry is
  always shown.
- **Preset loads:** ML Studio's prepared instance. An instance whose library is not among the listed plugins takes the
  state in place instead of being refused (`plugins::NO_LIBRARY`); one whose library changed is still refused.
- **Guide:** topics added for the seven AQUILA-only effects, which the guide's own test requires.
- **Help > About** stays AQUILA's (`MenuCmd`), not ML Studio's Settings page.
- `rack_additions_fit_channels...` allows the rack's top to sit a fraction of a pixel above the workspace's, because
  AQUILA keeps windows on whole pixels.

### Test status

`newfail.sh --workspace --lib --bins --tests --no-fail-fast`: 3120 passed, 100 failed, 140 ignored. Outside the
baseline: the 15 of section 11 plus `app::tests::transport_clock_digits_share_a_bottom_line`, which also fails in a
clean build of ML Studio v2026.11.13.2 on this machine.

### Still open

- Windows-only code is not compiled.
- `packaging/github-release.json` lists ML Studio's v2026.11.13.2 binaries; there are no AQUILA builds.
- The sampler's arpeggiator and echo were not listened to in the running app.
