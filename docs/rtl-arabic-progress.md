# RTL / Arabic editing — fork progress & continuity

**Status:** display, inline-editor joining, applying edits, and moving edited boxes all work on the
real test file; all code green and **committed** on `rtl-arabic-editing` (6 commits, not yet
pushed). · **Last updated:** 2026-10-20.

Fork: **zaidshurbaji/pdfcraft** (upstream: storytold/pdfcraft). Branch: **`rtl-arabic-editing`**.
Goal: make editing existing **right-to-left (Arabic/Hebrew)** text in PDFs work — it was
displaying reversed, letters disconnected, and refusing to apply edits.

## Current state (read first for a new session)
- On branch `rtl-arabic-editing`, **6 commits made, not yet pushed** (working tree now clean). The
  commits, in order:
  1. `feat(vendor): bidirectional text layout (UAX #9) in vendored epaint` — §1 (upstreamable unit:
     `vendor/epaint/**`, the `[patch.crates-io]` entry, `vendor/README.md`, the fonts.rs bidi tests)
  2. `feat(edit): apply edits to existing right-to-left text` — §2 apply fallback + §3
     variable→static flattening (`embedding::static_instance`) + §6 row-unification (all of
     `text_regions.rs`; `embedding.rs` split so only `static_instance` is here)
  3. `feat(ui-egui): join Arabic in the inline editor with the bundled shaper face` — §4
  4. `fix(edit): a degenerate object no longer blocks the page, and an edited RTL box can be moved`
     — §5 (`objects.rs`) + §7 (`text.rs`) with both revert-proven regressions
  5. `feat(fonts): drop the open-licence allowlist for document-input fonts (fork policy)` — the fork
     embedding policy (§8: `validate_document_input` + its revert-proven test), kept its own commit
     because it is NOT upstreamable
  6. `docs: RTL/Arabic editing progress and continuity doc` — this file
- Gates (all re-run green this session with `CRAFT_FONTS_DIR` set before committing): `cargo fmt
  --check`, `cargo xtask layers`, `cargo xtask assets`, clippy clean on `fonts`/`edit`/`ui-egui`;
  tests — fonts **75**, edit **188**, ui-egui `--test fonts` **13**, ui-egui `--lib edit_text_ui`
  **9**. Commit 2's snapshot was additionally checked standalone (the only intra-file split).
- Verified in the running app on `aqaba.pdf` (earlier this session): RTL display order,
  inline-editor joining, applying an edit to an Arabic paragraph, and moving an already-edited
  Arabic box.
- Next: **push the branch + open the PR** (fork → may stay fork-only; the upstreamable commits are
  1–4, commit 5 is fork-only), then **auto-use installed matching face** (STILL TODO #1). Candidate
  follow-up: `add_content` embeds a *variable* craft face, which blocks later edits of added Arabic
  (separate from everything above).

## Environment / how to build & run

- Repo: `C:\Zooz\Zooz\PDFcraft`
- **craft-fonts** (optional build input with the Noto Arabic/Hebrew/Thai faces) is cloned at
  `C:\Zooz\Zooz\craft-fonts`. It is **required** to build in the RTL fallback fonts.
- Build (release GUI):
  ```bash
  CRAFT_FONTS_DIR="C:/Zooz/Zooz/craft-fonts" cargo build --release -p pdfcraft
  ```
  **Kill the running app first** — a running `pdfcraft.exe` locks the binary and the link step
  fails with "Access is denied (os error 5)":
  ```bash
  powershell -Command "Get-Process pdfcraft -ErrorAction SilentlyContinue | Stop-Process -Force"
  ```
- Binary: `target/release/pdfcraft.exe` (portable, ~94 MB; no install needed).
- Tests (need craft-fonts for the RTL ones):
  ```bash
  CRAFT_FONTS_DIR="C:/Zooz/Zooz/craft-fonts" cargo test -p pdfcraft-ui-egui --test fonts
  CRAFT_FONTS_DIR="C:/Zooz/Zooz/craft-fonts" cargo test -p pdfcraft-edit  script_fallback_face
  CRAFT_FONTS_DIR="C:/Zooz/Zooz/craft-fonts" cargo test -p pdfcraft-fonts static_instance
  ```
- Real test file: `C:\Users\zaid-\OneDrive\Desktop\العقبة.pdf` (copied to the session scratchpad
  as `arabic_test/aqaba.pdf`).

## Root causes found

1. **Display "flips backwards"**: egui/epaint 0.36 has **no bidi** (its code literally says "need
   script-aware splitting once RTL/bidi support is added"). It shapes Arabic within a run (harfrust)
   but never reorders runs for right-to-left, and its cursor model (`Row::char_at`/`x_offset`)
   assumes glyph-index == logical-char-index with monotonic x.
2. **epaint RTL glyph-duplication bug**: the shaper returns RTL runs in *visual* order (clusters
   decreasing); epaint's continuation-glyph logic assumes clusters increase, so it emitted spurious
   zero-advance "continuation" glyphs for every RTL run, breaking `glyphs.len() == char_count`.
3. **Apply refused**: editing existing complex text needs the *complete* source face, but PDFs
   embed **subsets** → `shape_source` errored "complex text needs the complete source face".
4. **Variable-font embedding**: the bundled Noto fallback faces are **variable fonts**; embedding
   rejects variable fonts ("select a static instance…").
5. **Font policy stricter than Acrobat**: PdfCraft only embeds fonts that declare an **open
   license** (OFL/Apache/…), so installed proprietary fonts (Arial, Sakkal Majalla, Dubai, …) are
   excluded even when `fsType` permits embedding. Acrobat gates on `fsType` only.

## Approach chosen

**Approach B**: add real bidi to a **vendored, patched `epaint`** (gated on RTL so LTR/CJK are
byte-identical; intended to be upstreamable), plus edit-side fallbacks. Vendoring follows the
repo's existing `vendor/` + `[patch.crates-io]` convention (see `vendor/README.md`).

## What is DONE (and tested)

### 1. epaint bidi patch — `vendor/epaint/` (+ `[patch.crates-io] epaint` in `Cargo.toml`)
- `src/text/text_layout.rs`:
  - `Paragraph::glyph_bytes` — per-glyph logical byte, filled at every glyph emission
    (`layout_shaped_run`, `emit_continuation_glyphs`); `ShapingContext::run_base` tracks the
    absolute byte base.
  - **RTL run fix**: `layout_shaped_run` detects an RTL run (first cluster > last cluster) and
    iterates the shaped glyphs in **reverse (logical) order**, so clusters increase and the
    continuation logic emits exactly one glyph per character. This fixed the duplication bug.
  - `bidi_reorder_row` (UAX #9 via `unicode-bidi`) + `is_rtl_char` + `row_visual_width`: each
    visual line is reordered — glyphs kept in **logical order** with **visual `pos.x`** — wired
    into all three row-construction sites in `rows_from_paragraphs` / `line_break`. Engages only
    when the line has RTL script.
  - New dep: `unicode-bidi` (added to `vendor/epaint/Cargo.toml`).
- `src/text/text_layout_types.rs`: `Row::char_at` / `x_offset` made **bidi-aware** (caret + click),
  gated behind `Row::is_bidi()` so pure-LTR rows take the exact upstream path. Direction inferred
  geometrically via `Row::glyph_is_rtl`.
- Documented in `vendor/README.md` (table row + paragraph).
- Tests: `crates/ui-egui/tests/fonts.rs` →
  `arabic_text_lays_out_right_to_left_in_logical_order`, `latin_text_layout_is_unchanged_by_bidi`.
  **All 13 fonts tests pass**, CJK/Telugu/Latin unaffected.
- NOTE: fonts that encode combining marks as separate glyphs (Noto's i'jam dots) legitimately give
  `glyphs > chars`; marks sit on their base (zero advance, same x). egui's cursor model is
  imprecise for such marks — pre-existing, not newly broken.

### 2. Apply fallback — `crates/edit/src/text_regions.rs`
- `shape_source` now, when no document face covers the run, falls back to the bundled craft face
  for the run's script (`script_fallback_face`: Arabic→`document_arabic_font`,
  Hebrew→`document_hebrew_font`, Thai→`document_thai_font`). The whole run is shaped with that one
  face (correct joining, no mixed-face run). Test: `fallback_tests::script_fallback_face_picks_the_right_bundled_script`.

### 3. Variable→static flattening — `crates/fonts/src/embedding.rs`
- `static_instance(bytes)`: drops variation tables (`fvar/gvar/avar/cvar/HVAR/VVAR/MVAR/STAT`),
  rebuilds the sfnt; the default-instance outlines in `glyf` + cmap/GSUB/GPOS are kept. The edit
  fallback flattens the variable Noto face before shaping/embedding. Test:
  `static_instance_flattens_a_variable_font_so_it_embeds_and_still_shapes`.

### 4. Live editor joining (Stage 3b) — `crates/ui-egui/src/edit_text_ui.rs`
- The inline editor rendered existing Arabic with the PDF's embedded *subset* font, which usually
  ships no GSUB table, so the letters showed in their **isolated (disconnected)** forms while
  typing — even though the bidi work already put them in the right order. Root cause: the
  interface registers the bundled Noto Arabic face only as a *fallback*, but the subset (which
  holds the isolated glyphs) wins as the *primary* face, so egui/harfrust never reaches the
  shaping-capable fallback.
- Fix:
  - `install_source_fonts` now registers a named family `pdf-source-arabic` =
    `[document_arabic_font().name(), …Proportional fallback]` (helper `arabic_shaper_stack`),
    guarded to a no-op without craft-fonts.
  - `editor_job` detects Arabic in each run (`has_arabic`) and, for an unchanged original face,
    displays that run with the shaping family instead of the subset — so existing text *and*
    newly typed letters join consistently with one face. Justification letter-spacing is dropped
    for those runs (advance between joined glyphs would split them) and synthetic italics are
    suppressed. Mirrors the apply-side `text_regions::shape_source` fallback, so display and the
    saved edit use the same craft face.
- Tests (`crates/ui-egui/src/edit_text_ui.rs` `mod tests`): `has_arabic_matches_arabic_script_only`,
  `arabic_shaper_stack_leads_with_the_bundled_face`, `the_shaper_family_joins_arabic` (joining
  shown as narrower than ZWNJ-separated, mirroring the Telugu test), and end-to-end
  `the_editor_lays_out_an_arabic_run_with_the_shaper_face` (reverting the `editor_job` override was
  verified to fail it: it picked the subset `pdf-source-0` instead of `pdf-source-arabic`).
- NOTE: Hebrew and Thai keep their usual faces — Hebrew doesn't join (order alone was the bug) and
  Thai is out of this change's scope; the reported symptom was Arabic joining. Auto-use of an
  *installed* matching family (fuller Acrobat parity) is still a later step.

### 5. One degenerate object no longer blocks the whole page — `crates/edit/src/objects.rs`
- Found while testing moves on `aqaba.pdf`: the object-select tool refused *"These objects could
  not be moved…"*. Root cause (via the CLI `object_list`, which keeps the detailed error):
  `editable_objects` errored *"an object has invalid geometry"* because the page has a zero-width
  text box — one undecodable glyph with zero advance (`rect=[240.196, 279.135, 240.196, 295.791]`,
  text `"\u{fffd}"`). A single such junk placement made the entire page unselectable, unlistable
  **and unmovable** (the move re-inventories the page, so even moving a valid block failed).
- Fix: `editable_objects` now **drops** objects whose rect fails `valid_rect` (zero/negative
  area, non-finite, or out of bounds) instead of erroring. Such an object can't be hit or dragged
  anyway. Per-kind indexes come from the per-kind listings, so dropping one keeps every other
  object's index, and `move_objects` (same inventory) stays consistent. Not RTL-specific, but it
  unblocks moving Arabic text boxes on real documents.
- Test: `crates/edit/src/tests.rs::a_zero_geometry_object_does_not_block_selecting_or_moving_the_page`
  (a `Widths [0]` glyph beside a valid paragraph; verified to fail when reverted — it errored
  "an object has invalid geometry"). Confirmed end to end on `aqaba.pdf`: `object_list` returns all
  32 objects and `object_move` on an Arabic block succeeds.
- Also fixed a pre-existing clippy `single_range_in_vec_init` in `script_fallback_face` (Thai
  branch) so the edit crate is clippy-clean.

### 6. Row-unification for mixed contextual runs (insert-new-char) — `crates/edit/src/text_regions.rs`
- Found while testing edits on `aqaba.pdf`: editing an Arabic paragraph raised *"contextual/bidi
  text across font or style boundaries needs a uniform full-face row; mixed contextual runs are not
  yet supported"*. A visual row mixes runs when existing subset glyphs sit beside a character that
  fell back to another face (a newly typed letter the subset lacks, or two adjacent source runs in
  different subsets). `rewrite_block` refused outright.
- Fix: `rewrite_block` now **unifies** such a row — for a non-kept row whose text needs shaping and
  that mixes `(source line, resource)`, it routes the whole row onto one face that can shape all of
  it (a covering document face if one exists, otherwise the bundled script fallback, both resolved
  by `shape_source`), by setting every character in the row to the anchor's line and resource. The
  row then shapes as a single run, so joining and right-to-left reordering span it. The edit is
  refused only when nothing can shape the row (then `shape_source`'s own error propagates).
- Verified end to end on `aqaba.pdf` via the CLI (`text_paragraphs` → `text_edit`): inserting an
  Arabic letter into the multi-run paragraph 15, which previously failed, now applies and reshapes
  on `NotoSansArabic`. No committed unit regression: the edit-crate test harness cannot synthesize a
  **static** Arabic subset source (its `add_content` embeds the *variable* face, so any later edit
  fails at `embed_mapping` on the variable font — unrelated to this change; real subsetted PDFs ship
  static subsets). Reproduce from the repo root:
  ```
  # list.json: [{"tool":"doc_open","args":{"path":"aqaba.pdf"}},{"tool":"text_paragraphs","args":{"doc":1,"page":1}}]
  # then text_edit with {"doc":1,"page":1,"paragraph":15,"text":"<editor_text>ة"} — status ok after the fix.
  ```
- NOTE: unifying sets the whole row to the anchor char's source line, so per-line size/overrides for
  other runs in that row are not preserved; acceptable (the old code refused the edit entirely). A
  separate, pre-existing issue surfaced: `add_content` can embed a *variable* craft face, which then
  blocks editing — a candidate follow-up, out of scope here.

### 7. Moving an edited paragraph — `crates/edit/src/text.rs` (`translate_blocks`)
- Found while testing: after editing an Arabic box, moving it failed with *"These objects could not
  be moved…"*. Root cause (via CLI diagnostic): the byte-preserving group move required **every**
  operator a line owns to be a movable show operator (`patches.len() != selected.len()`). A shaped
  right-to-left rewrite wraps its glyphs in a `/Span <</ActualText …>> BDC … EMC` scope, and
  `text_lines` folds that span's `BDC`/`EMC` into the line's operators — so an edited Arabic line
  owns 2 non-show ops beside its `Tj`s (observed: `patches=12 selected=14`), and the move was
  refused.
- Fix: `translate_blocks` now refuses only when a **show** operator the line owns could not be moved
  (`is_show` filter), leaving the owned state/structural operators (BDC/EMC, font, rise, spacing,
  positioning) in place — their effect is preserved by the TJ/Ts wrappers around each show.
- Test: `crates/edit/src/tests.rs::a_paragraph_in_an_actualtext_span_can_be_moved` (a Latin
  paragraph inside a `/ActualText` span, no craft-fonts needed; verified to fail when reverted:
  "a selected text operation cannot be moved"). Confirmed end to end on `aqaba.pdf`: editing an
  Arabic paragraph then moving it now succeeds.

### Verified in the app (`aqaba.pdf`, this session)
- RTL **display order** correct, and the inline editor shows Arabic **joined** while typing (§4).
- **Applying** an edit to an Arabic paragraph succeeds, including inserting a letter the subset
  lacks (§6 row-unification).
- **Moving** an Arabic box works, including one that was **already edited** (§5 and §7).

### 8. Acrobat-style embedding policy (fork-only) — `crates/fonts/src/embedding.rs`
- Upstream's `validate_document_input` required an open-license **name** (OFL/Apache/…) on top of
  the embeddability checks. This fork **drops that allowlist**, so installed proprietary faces
  (Arial, Sakkal Majalla, …) become usable via *Installed fonts…* / *Load font…* and are listed by
  `discover()` (`crates/engine/src/text_fonts.rs`), like Acrobat.
- IMPORTANT finding (corrected 2026-10-20): the font's own OS/2 **`fsType`** embedding permissions
  were **already** enforced upstream — `validate_document_input` calls `names()`, which calls
  `validate()`, which rejects restricted-license (`0x0002`), preview/print-only (`0x0004`) and
  bitmap-embedding-only (`0x0200`) faces. (No-subsetting `0x0100` is allowed because PdfCraft embeds
  the **full** program, never a subset.) So the first draft's extra `fsType` check *inside*
  `validate_document_input` was **redundant dead code** and has been removed; the fork change is now
  narrowly "drop the open-license allowlist". The two conservative asset-policy guards are kept: the
  Source-Han family exclusion and the Adobe-authored (copyright/manufacturer) exclusion.
- This is a **fork policy choice**, NOT upstreamable (upstream deliberately requires open licenses).
- Test (revert-proven): `embedding::tests::document_input_no_longer_requires_an_open_licence_name` —
  strips Inter's license-description name record and asserts the otherwise-valid, embeddable font
  now validates; also checks that `fsType`-restricted faces are still refused (via `validate`).
  Restoring the allowlist was confirmed to fail it at "a valid embeddable font with no open-licence
  name must validate".

## NEXT

- **Push the branch and open the PR** (fork-first; see below for the upstreamable split).

## STILL TODO (not started)

- **Auto-use installed matching face** (fuller Acrobat parity): before the Noto fallback,
  `shape_source` could try an installed font matching the source family. Caveat: `discover()` scans
  dirs and is done off-thread on request today — doing it synchronously in `shape_source` may be
  slow; needs a cached/off-thread design.
- **`add_content` embeds a variable craft face** (candidate follow-up, surfaced in §6): adding Arabic
  via `add_content` embeds the *variable* Noto face, so a later edit of that added text fails at
  `embed_mapping` ("select a static instance…"). Likely wants the same `static_instance` flattening
  the edit path uses. Out of scope for the fixes above; real subsetted PDFs are unaffected.

## Key files touched
- `Cargo.toml` — `[patch.crates-io] epaint = { path = "vendor/epaint" }`
- `vendor/epaint/**` — vendored + bidi patch; `vendor/epaint/Cargo.toml` adds `unicode-bidi`
- `vendor/README.md` — epaint patch documented
- `crates/edit/src/text_regions.rs` — `shape_source` fallback + `script_fallback_face` + test
- `crates/edit/src/objects.rs` — `editable_objects` drops invalid-geometry objects instead of
  refusing the page (+ `tests.rs` regression)
- `crates/edit/src/text.rs` — `translate_blocks` moves a line's show ops only, so an edited
  paragraph (which owns `/ActualText` BDC/EMC) can be moved (+ `tests.rs` regression)
- `crates/fonts/src/embedding.rs` — `static_instance` + test (commit 2); `validate_document_input`
  fsType policy (commit 5, §8)
- `crates/ui-egui/src/edit_text_ui.rs` — Stage 3b editor joining: `ARABIC_SHAPER` family, `has_arabic`,
  `arabic_shaper_stack`, `editor_job` run override + tests
- `crates/ui-egui/tests/fonts.rs` — bidi layout tests

## Revision history
- 2026-10-20 (latest) — Closed the §8 test gap: found the first-draft `fsType` check in
  `validate_document_input` was redundant (embeddability is already enforced by `validate()` via
  `names()`), so narrowed the fork change to dropping the open-licence allowlist and added a
  revert-proven test (`document_input_no_longer_requires_an_open_licence_name`). Commits 5 and 6
  rebuilt to reflect this. fonts crate clippy-clean + tests green.
- 2026-10-20 (later) — Committed all work as 6 commits on `rtl-arabic-editing` (breakdown in Current
  state); embedding policy (§8) finished and committed on its own. All gates re-run green. Not yet
  pushed; PR pending.
- 2026-10-20 — Stage 3b editor joining (§4); degenerate-object move fix (§5) + Thai clippy cleanup;
  row-unification for mixed contextual runs (§6); moving an edited paragraph (§7). Added current-state
  summary and this history. All edit-crate gates green; app verified on `aqaba.pdf`.
- 2026-10-19 (and earlier) — epaint bidi patch (§1), apply fallback (§2), variable→static flattening
  (§3); fork embedding policy started (IN PROGRESS).
