# The SSOT an integration surface needs

Cosmograph is getting more front doors: a Streamlit component, a Dash component, the Jupyter widget that already exists, and an MCP App that Claude drives. Each of them has to answer the same question — *what parameters does Cosmograph take, what are they, and which of them is this user allowed to point at a column of their dataframe?* — and each of them is currently free to answer it by guessing from a name suffix or by hardcoding a list. This document says what those surfaces need from the generated parameter SSOT, what the SSOT provides today, and what is missing.

It is written for two readers: whoever builds a surface, and whoever changes the generator that feeds them.

## The artifact, and what you may depend on

The parameter SSOT is generated from Cosmograph's TypeScript sources by `pnpm run ai:params-ssot` in the `cosmograph` repo (added in cosmograph#631). It is committed at `packages/cosmograph/ai/params-ssot.json` and, because the `ai:copy` step copies all of `ai/` into `dist/`, it also ships inside the published `@cosmograph/cosmograph` npm package. That is the artifact to depend on.

**Depend on the file, not on a Python import.** `cosmograph.util._params_ssot()` in the Python package is a *derived* view: it is the subset of parameters the Jupyter widget's traitlets expose, carrying Python type annotations. It is the right thing for `cosmo()` and the wrong thing for a surface that is not the Jupyter widget, because it silently drops everything the widget has no traitlet for, and it is regenerated on the Python side's own schedule.

**Stability contract.** Fields are added, not renamed or removed, without a note in this document. The generator runs in the cosmograph repo's CI: `tests/unit/params-ssot.test.ts` regenerates the artifact in memory and fails the build when the committed copy no longer matches its TypeScript sources, so a silent drift is not possible. What *is* possible is an intentional upstream change — a parameter renamed, a default moved — landing between two versions you pinned.

**Versioning.** The document carries `$id`, `title`, `description`, a `source` block naming the TypeScript files it came from, and — as of cosmograph#633 — a `version` taken from the package version. Pin against that. Before #633 lands, the only version handle is the npm package version you fetched the file from.

## What every surface needs, per parameter

These are the questions a surface asks about a parameter. The first four are the ones that decide what widget to render at all; the rest decide how to render it.

**`binding` — what kind of thing is this?** The load-bearing field, and the one no surface should have to infer. Emitted as of cosmograph#633, derived from the TypeScript *type* rather than the name. Five values:

- `data` — the `points` or `links` table itself. Not a control at all; this is where the user's dataframe goes.

- `column` — the parameter takes the *name of a column* in the user's data. `pointColorBy`, `linkSourceBy`, `pointXBy`. The surface renders a column picker populated from the loaded dataframe, not a text box. There are 23 of these plus the two `*IncludeColumns` list parameters.
- `value` — the parameter takes a literal. `pointSizeScale`, `backgroundColor`, `simulationRepulsion`. The control comes from the JSON-Schema type fragment: boolean → switch, enum → select, number → slider or number input, string with a colour-shaped description → colour picker.
- `function` — the parameter takes a JavaScript callback that computes a value per row: the seven `*ByFn` accessors (`pointColorByFn`, `linkWidthByFn`, and so on). **A surface must skip these deliberately and visibly**, not drop them silently. A Python or MCP surface cannot serialise a JS closure; a user who came looking for `pointColorByFn` deserves to be told it is JS-only and pointed at `pointColorBy` plus `pointColorByMap`.
- `event` — the parameter is an *output*, not an input: the 40 `on*` callbacks. Rendering one as a control is always a bug. These belong to the event side, described further down.

**`target` — `point`, `link`, or `global`.** Panel grouping. Trivial for the generator to know and guessy for a consumer to reconstruct.

**`group` — `styling`, `simulation`, `labels`, `layout`, `interaction`, `data`.** Ordering within a panel. A surface that renders 228 controls in declaration order is unusable; a surface that renders six labelled sections is not.

**`reactive` — can this change after the graph is built?** The difference between a live slider and a control that must tear down and re-mount the visualisation. `enableSimulation`'s own description says it "will be applied only on component initialization and it can't be changed using the setConfig method" — in prose, in a sentence, which is not something a surface can switch on.

**Type, enum and range.** Already present, as a JSON-Schema fragment per parameter under `schema`, with `$ref`s resolving against a `definitions` block in the same file. Eight parameters resolve to string enums — the strategy parameters (`pointColorStrategy`, `pointSizeStrategy`, `linkColorStrategy`, `linkWidthStrategy`), `pointLabelPosition`, `spaceDimensions`, `componentsDisplayStateMode`, `statusIndicatorMode` — and those are exactly the ones that should render as a select rather than a text field. Numeric *ranges* are not in the schema; where a sensible range exists it is usually stated in the description prose.

**Default.** Present for the 43 parameters that `defaultCosmographConfig` sets. Absent means the library decides at runtime, which is not the same as "no default" — a surface should render such a control as unset rather than inventing a value.

**Description.** Present for nearly every parameter, taken from the TypeScript JSDoc. It is written for a developer reading an IDE tooltip, so it is accurate but often mentions the camelCase name of a sibling parameter. A surface showing it to an end user should expect that.

**`pythonName`.** The snake_case name, emitted as of cosmograph#633. Mechanical from camelCase — except where it is not: `showFPSMonitor` is `show_fps_monitor`, and a naive regex produces `show_f_p_s_monitor`. That is one exception out of 125 today, which is exactly the kind of ratio that gets a hand-rolled converter shipped and then quietly broken by the second acronym.

**`layer` — engine or library?** Which renderer implements the parameter, and the field that says whether a control will do anything at all in the surface you are building. Not emitted yet; see the proposed changes below.

## A parameter does not mean the same thing in every renderer

**This section is provisional.** It holds if the MCP App surface renders Cosmograph directly inside the app iframe. A nested-iframe route is being tested — a full-Cosmograph embed hosted on its own origin with its own CSP, reached through `frameDomains` and driven over a postMessage bridge — and if that works the MCP surface gets the full library back, `binding: "column"` means one thing again everywhere, and `layer` drops from a correctness requirement to a renderer hint. Do not build a control strategy on this section until that result is in.

There are two renderers, not one, and a surface may not get to choose which it uses.

`@cosmograph/cosmograph` is the full library: it carries duckdb-wasm and the data-kit, and it resolves a column name against the user's data in the browser. `@cosmos.gl/graph` is the engine underneath it, which takes typed arrays and knows nothing about columns or tables.

The MCP App surface is forced onto the engine. Under the Content Security Policy the MCP Apps specification tells hosts to apply, blob-backed Workers are refused — `worker-src` is unset, so it falls back to `script-src`, which does not allow `blob:` — and `WebAssembly.instantiate` is refused because `'unsafe-eval'` is not granted. duckdb-wasm needs both, and the CSP declaration a resource can request has no field for either, so this is structural rather than a configuration mistake. The engine has no wasm and no workers and runs there unchanged.

The consequence for anything generated from this SSOT is that `binding: "column"` means two different things depending on the renderer. On the library, `pointColorBy` is a column name and the library resolves it. On the engine, the same visual result is a `Float32Array` the surface has to compute itself, in Python, before it hands anything over. A surface that generates a column picker from `binding` alone and then renders on the engine produces a control that silently does nothing — the same failure mode as an init-only parameter treated as reactive, and just as hard for a user to tell from a bug in their data.

`layer` is what closes that, and the partition exists in exactly one place: `CosmographConfig` extends `Omit<GraphConfig, ...>` from cosmos, so generating a schema for cosmos' `GraphConfig` alone marks the engine half precisely. Nobody downstream can reconstruct it.

One shape detail while you are writing example calls: `new Cosmograph(el, config)` takes a required config — omitting it throws — and `prepareCosmographData` takes a nested `{ points: {...}, links: {...} }` preparation config, not a flat `CosmographConfig`.

## The event and output side

Parameters are only half of what a surface needs. The other half is what comes *back*: which points the user selected, where the simulation put them, what is under the cursor.

Two different mechanisms carry this today, and a surface author should know which one they are looking at.

**The 40 `on*` callbacks in `CosmographConfig`** are the JavaScript-native event surface: `onClick`, `onPointsFiltered`, `onSimulationEnd`, and so on. They appear in the params SSOT with a `$comment` giving the TypeScript signature, and their argument shapes are in the schema under `namedArgs`. A surface that runs JavaScript (the MCP App, a Dash component with a JS bundle) can wire these directly. A surface that does not must bridge them.

**The Jupyter widget's output traitlets** are how the Python side already bridges them: `clicked_point_index`, `clicked_point_id`, `clicked_cluster`, `selected_point_indices`, `selected_point_ids`, `selected_link_indices`. These are widget state that changes in the browser and syncs back to Python. They are *not* in the params SSOT, because they do not exist in `CosmographConfig` — they are an invention of the widget layer.

That gap is worth naming plainly: **there is no SSOT for the event side.** A Streamlit or Dash component that wants selection back has to reimplement the same bridge the Jupyter widget already has, and the two will drift. Defining it is a larger piece of work than this document covers, but the shape is clear — for each event, the JS callback it comes from, the payload, and the name the payload is surfaced under on the Python side. Until that exists, treat the widget's traitlet names as the de-facto contract and copy them, so that at least the surfaces agree with each other.

Point positions are a third case again: they are neither a config parameter nor an event, but a result you have to ask the running instance for.

## The data contract, which is not in this SSOT

The params SSOT describes parameters. It does not describe the tables and columns those parameters point *into*, and every surface needs both.

The canonical names live in `packages/cosmograph/src/enum.ts` in the cosmograph repo: the prepared tables are `cosmograph_points` and `cosmograph_links`, and the reserved column names are `id`, `idx`, `points_idx_seq`, `source`, `sourceidx`, `target`, `targetidx`. Those are the names Cosmograph generates and expects; a user column that collides with one of them is a problem a surface should catch early.

Which parameters are *required* is machine-readable in `packages/cosmograph/src/cosmograph/config/interfaces/data.ts`, in the `RequiredPointsConfigKeys` and `RequiredLinksConfigKeys` const objects: points need `points`, `pointIdBy` and `pointIndexBy`; links need `links`, `linkSourceBy`, `linkSourceIndexBy`, `linkTargetBy` and `linkTargetIndexBy`. A surface can use that to decide when it has enough to render at all, and what to prompt for when it does not.

`packages/cosmograph/ai/schemas/cosmograph-data-prep-config.schema.json` — generated by the same build step — describes the data-preparation config, including every `*By` mapping field and the strategy enums. It is the closest thing to a machine-readable data contract that exists today. Consume that rather than hardcoding `enum.ts`, and treat the reserved-name list as the one piece you still have to mirror by hand.

## What each surface actually does with this

**Streamlit.** One render pass, no persistent instance; the whole config is rebuilt from Python on every interaction. Needs `binding` to decide picker versus control, `group` and `target` to lay out the sidebar, and `reactive` least of all — everything is a re-render anyway. Needs the event bridge most, because Streamlit's model is that the component returns a value.

**Dash.** Callback-driven, with a long-lived component. `reactive` matters here: a parameter that can only be set at construction must either be excluded from the live controls or trigger a re-mount, and getting that wrong looks like a control that silently does nothing. That is the single worst failure mode for a generated UI, because the user cannot tell it from a bug in their data.

**The Jupyter widget.** Already exists and already consumes the derived Python view. What it gains from this SSOT is the enum and range information it currently does not use, and a path away from hand-maintaining `js/config-props.json`.

**The MCP App.** An agent, not a person, is choosing parameters. It needs the descriptions verbatim, the enums to avoid inventing strategy names, and `binding` to know which arguments must be column names from the user's actual data rather than free strings — that distinction is where an agent most reliably hallucinates. It benefits from the full 228, where a human UI benefits from a curated subset.

## Proposed generator changes

Small, additive, one concern each. In the order they are worth doing.

1. ~~**`version` on the document.**~~ Done in cosmograph#633, taken from `packages/cosmograph/package.json`.
2. ~~**`binding` per parameter**, derived from the TypeScript type rather than the name.~~ Done in cosmograph#633. Deriving from the type rather than the `By` suffix paid for itself immediately: it catches `pointLabelFn` and `pointLabelWeightFn`, which a suffix rule misses, and it leaves `string | function` unions such as `pointLabelClassName` as `value` so a surface still offers the string.
3. **`target` and `group`**, from the interface that declares each parameter. `CosmographPointsConfig`, `CosmographLinksConfig`, `SimulationConfig`, `LabelsCosmographConfig`, `BasicConfig`, `CallbackConfig` and cosmos' `GraphConfig` already partition the parameters exactly the way a panel layout wants them; the generator loses that information today by flattening `CosmographConfig` into one property bag.
4. ~~**`pythonName`**, emitted rather than derived, for the acronym reason above.~~ Done in cosmograph#633.
5. **`layer`**, `engine` or `library`, from whether the parameter comes from cosmos' `GraphConfig`. Worth having either way — it tells a surface which half of the API it is on — but whether it is *load-bearing* depends on the provisional section above. If the MCP surface keeps the full library, this is a hint rather than a correctness requirement.
6. **`reactive`**, which needs a decision from the JS side before the generator can do anything: there is no machine-readable marker for init-only parameters today, only prose. The cheapest fix is a JSDoc tag — `@initOnly` — on the parameters that cannot go through `setConfig`. Until that exists the generator should emit nothing rather than a guess, and surfaces should treat a missing `reactive` as unknown rather than as `true`.

Everything except `reactive` is derivable from what the TypeScript already says. `reactive` is a request to the library, not to the generator, and should be raised as such.

## Where this fits

- The generator: `packages/cosmograph/scripts/generate-params-ssot.mjs` in the cosmograph repo, run by `pnpm run ai:params-ssot`.
- The artifact: `packages/cosmograph/ai/params-ssot.json`, also shipped in `@cosmograph/cosmograph`.
- The Python consumer: `cosmograph/_dev_utils/params_ssot.py` in py_cosmograph refreshes `cosmograph/data/params_ssot.json` from it.
- The freshness gates: `tests/unit/params-ssot.test.ts` in cosmograph, `tests/params_ssot_test.py` in py_cosmograph.
