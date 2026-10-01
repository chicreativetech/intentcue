# Visual Review Tool — Build Spec

Sep 30, 2026 · @Jonas Bergdahl Chi

## Overview

Build a lightweight visual review layer between a human and a coding agent: the agent builds app screens, the tool shows them as tiles, the human annotates them, and the tool writes precise, file-based instructions any agent can read.

The tool never edits the app and contains no AI. It supplies context; the coding agent supplies the intelligence. The working name is not decided; this spec uses `review` for the CLI command and `.review/` for the project folder as placeholders.

**Users.** Designers and developers who build apps with coding agents (Claude Code, Codex, Cursor and similar) and want to give visual, spatial feedback instead of describing changes in text.

**Principles**

- **Point, don't describe.** Every annotation resolves to a concrete UI element or an explicit empty region.
- **Files are the contract.** All input and output is plain files in the project, readable by any agent and committable to git.
- **Platform-neutral core.** iOS, Android and web feed the same element schema; nothing past the capture layer knows the platform.
- **Deterministic output.** The same annotations always compile to the same instructions.
- **Small and fast.** One npm package, installed and started with one command.

**Non-goals**

- No visual editing of real elements: no moving, resizing, restyling or typography controls.
- No design system editor, component library, vector tools, auto layout or prototyping.
- No AI inside the tool in the MVP.
- No automatic discovery of screens or routes; the agent declares them.
- No native desktop app in the MVP; a Tauri wrapper is a later option.

## The review loop

Every round runs the same five steps, and the agent never changes UI code from a round the human has not sent.

&#91;embedded content: review loop · 5 steps, repeating per round\]

A round moves through the states `capturing`, `open`, `sent` and `applied` in `status.json`. After the agent applies a round, the next capture starts a new round; the first round is the only one that begins without feedback.

## Architecture and repository structure

Build a TypeScript monorepo with pnpm workspaces and five packages, shipped to npm as one package; `core` holds all product logic and has no I/O.

&#91;embedded content: package architecture · 5 packages\]

Arrows point from a package to what it uses. The CLI starts the server and runs capture; the server serves the built canvas; every package imports schemas from `core`.

```
packages/
  core/      schemas, resolver, compiler (pure functions)
  capture/   platform adapters + parsers
  server/    Hono server, WebSocket, .review/ file storage
  canvas/    React + Vite app
  cli/       entry point, commands, later the MCP server
fixtures/    captured hierarchies, screenshots, golden review output
examples/    sample app with Maestro flows for end-to-end tests
```

**Stack:** Node 20+, TypeScript strict mode, Zod, Vitest, Playwright, Hono, React 18+, Vite. Keep runtime dependencies minimal; no state library beyond React state and a small store (Zustand) in the canvas.

## The .review folder contract

The `.review/` folder in the project root is the only interface between the tool and the agent; everything the agent writes or reads lives there as plain files.

```
.review/
  screens.json          screen manifest, written by the agent
  rules.md              persistent design rules, appended by the tool
  flows/                Maestro flows or setup scripts per screen state
  rounds/
    001/
      review.md         compiled instructions for the agent
      review.json       the same, structured, with element targets
      annotations.json  raw annotations as drawn (for re-editing)
      screens/          <screenId>.png and <screenId>.annotated.png
      trees/            <screenId>.json, normalized element tree
      ink/              <annotationId>.png, cropped handwritten notes
      status.json       round state: capturing | open | sent | applied
  latest -> rounds/001  symlink or pointer file to the newest round
```

**Ownership.** The agent writes `screens.json` and `flows/`. The tool writes everything under `rounds/` and appends to `rules.md`. Humans may edit `rules.md` by hand; the tool must preserve those edits.

**Rounds are immutable once sent.** A new review always creates a new round folder, so rounds can be diffed and committed.

**Agent integration.** `review init` appends this section to `AGENTS.md` (and `CLAUDE.md` if present), creating the file if needed:

```markdown
## Visual design review

Design feedback lives in `.review/`.

- Before any UI work, read `.review/rules.md` and follow it.
- When `.review/latest/status.json` says `sent`, implement `.review/latest/review.md`,
  then set its status to `applied`.
- When you add, remove or change screens, update `.review/screens.json`.
- If an instruction is marked `unresolved`, ask the user instead of guessing.
- To request a review, run `review capture` and tell the user it is ready.
```

## Data model

All schemas live in `packages/core` as Zod schemas, exported as TypeScript types and as JSON Schema files, so the server, canvas, CLI and agents validate the same shapes.

### Element tree

Every capture adapter outputs this shape. Coordinates are in screenshot pixels, origin top-left.

```ts
type UIElement = {
  id: string;              // accessibility id / testID / DOM id, else a stable generated path id
  idSource: "accessibility" | "testId" | "dom" | "generated";
  type: string;            // normalized: button, text, image, input, list, container, ...
  nativeType?: string;     // original platform class, e.g. "XCUIElementTypeButton"
  label?: string;          // visible text or accessibility label
  bounds: { x: number; y: number; w: number; h: number };
  visible: boolean;
  children: UIElement[];
  source?: { file: string; line?: number; component?: string };
};

type ScreenCapture = {
  screenId: string;
  platform: "ios" | "android" | "web";
  device: { name: string; width: number; height: number; scale: number };
  screenshot: string;      // relative path to PNG
  root: UIElement;
  capturedAt: string;      // ISO timestamp
};
```

Generated ids must be stable across captures of an unchanged screen: derive them from the path of types and sibling indexes, plus label when present.

### Screen manifest

`screens.json` is written by the agent and lists what to capture.

```json
{
  "version": 1,
  "app": { "name": "Checkout demo", "platform": "ios", "bundleId": "com.example.checkout" },
  "screens": [
    {
      "id": "checkout-default",
      "title": "Checkout",
      "group": "Purchase flow",
      "flow": "flows/checkout-default.yaml"
    },
    {
      "id": "checkout-card-error",
      "title": "Checkout, card declined",
      "group": "Purchase flow",
      "flow": "flows/checkout-card-error.yaml"
    }
  ]
}
```

For web, a screen entry uses `url`, optional `viewport` and optional `setup` script path instead of `flow`.

### Annotations

Raw drawings as the human made them, stored in `annotations.json`. Geometry is in screenshot pixels.

```ts
type Annotation = {
  id: string;
  screenId: string;
  kind: "comment" | "circle" | "arrow" | "rectangle" | "remove" | "freehand" | "rule";
  geometry:
    | { type: "point"; x: number; y: number }
    | { type: "path"; points: [number, number][] }
    | { type: "rect"; x: number; y: number; w: number; h: number }
    | { type: "arrow"; from: [number, number]; to: [number, number]; toScreenId?: string };
  text?: string;
  attachedTo?: string;           // id of another annotation this comment belongs to
  targets?: string[];            // element ids, set explicitly for rule annotations
  ink?: InkData;                 // pen strokes, see Pen input
  resolution?: Resolution;       // filled by the resolver, editable by the human
};

type Resolution = {
  status: "resolved" | "region" | "unresolved";
  elements: string[];            // element ids
  toElements?: string[];         // arrow target
  region?: { x: number; y: number; w: number; h: number };
  confirmedByUser: boolean;
};
```

### Review output

`review.json` holds compiled instructions; `review.md` is the same content as readable text.

```ts
type Instruction = {
  id: string;                    // "R3-5": round 3, instruction 5
  screenId: string;
  action: "remove" | "change" | "move" | "add" | "relate" | "note";
  targets: TargetRef[];
  destination?: TargetRef | { region: Rect };
  text: string;                  // human comment, verbatim
  instruction: string;           // compiled sentence
  status: "resolved" | "unresolved";
  annotationIds: string[];
};

type TargetRef = {
  elementId: string;
  type: string;
  label?: string;
  bounds: Rect;
  source?: { file: string; line?: number; component?: string };
};
```

## Capture adapters

Each adapter implements one interface and hides every platform detail from the rest of the tool; the MVP ships one mobile adapter, and the others follow behind the same interface.

```ts
interface CaptureAdapter {
  platform: "ios" | "android" | "web";
  check(): Promise<{ ok: boolean; problems: string[] }>;   // tools installed, device booted
  prepare(screen: ScreenEntry): Promise<void>;             // run flow / navigate / setup
  capture(screen: ScreenEntry): Promise<ScreenCapture>;    // screenshot + normalized tree
}
```

Adapters mostly spawn existing command-line tools and parse their output. Each parser gets fixture-based unit tests with real captured output checked into the repo.

| Adapter | Navigation | Screenshot | Element tree | Source mapping |
| --- | --- | --- | --- | --- |
| iOS simulator | Maestro flow | `xcrun simctl io booted screenshot` | `idb ui describe-all` (JSON) or Maestro hierarchy | React Native / Flutter inspector, later |
| Android emulator | Maestro flow | `adb exec-out screencap -p` | `adb shell uiautomator dump` (XML) or Maestro hierarchy | React Native / Flutter inspector, later |
| Web | Playwright `goto` + setup script | Playwright `screenshot` | DOM walk in page context | `data-component` attributes, React DevTools hook, later |

**Normalization rules**

- Convert all bounds to screenshot pixels, accounting for device scale (points vs pixels on iOS).
- Map native classes to the normalized `type` list; keep the original in `nativeType`.
- Drop invisible and zero-size elements; collapse wrapper containers that have exactly one child and identical bounds.
- Prefer ids in this order: accessibility identifier, testID, DOM id, generated.

**Doctor command.** `review doctor` runs `check()` for the configured platform and prints exactly which tool is missing and how to install it. Missing tools are the most likely first-run failure.

**Source mapping** is optional and out of MVP scope. When present, it fills `UIElement.source` so instructions can name the component and file.

## Canvas and annotation tools

The canvas is a React and TypeScript app built with Vite, with an SVG annotation layer over each screenshot; no canvas library is used in the MVP.

**Layout**

- **Overview:** an infinite, pannable and zoomable board with every screen as a tile, grouped by the manifest's `group` field. Tiles show title, platform and an annotation count.
- **Focus view:** double-click a tile to open it large, with the toolbar. Annotations can also be made directly on the overview.
- **Top bar:** project name, round number, screen count, unresolved count, and the **Send to agent** button.

**Tools**

| Tool | Key | Gesture | Needs text |
| --- | --- | --- | --- |
| Select | V | Click an element or annotation | No |
| Comment | C | Click to pin, then type | Yes |
| Circle | O | Drag a freeform loop | Optional |
| Arrow | A | Drag from start to end; may end on another tile | Optional |
| Rectangle | R | Drag a box where something should go | Optional |
| Remove | X | Click an element to strike it out | Optional |
| Freehand | P | Draw a free path | Optional |
| Rule | U | Shift-click several elements, on any screens, then type | Yes |

**Element layer**

- Hovering in Select, Comment or Remove shows the element under the cursor as an outlined box with its label and type.
- `Alt` cycles through nested elements under the cursor (child to parent).
- A toggle shows all element outlines at once, for checking capture quality.

**Resolution feedback**

- When a drawing ends, the resolved targets highlight for 1.5 seconds and a small chip under the annotation shows the target label, for example "Pay now button".
- Clicking the chip opens a picker: parent, children, or "empty area". A picked target sets `confirmedByUser: true`.
- Unresolved annotations get a warning chip and are counted in the top bar; the human can still send them.

**Editing annotations.** Every annotation can be selected, moved, deleted, and have its text edited. Undo and redo cover all annotation actions. Annotations autosave to `annotations.json` on every change.

**Visual style.** Annotations use one high-contrast accent colour, 2 px strokes and numbered badges matching instruction numbers. The UI stays quiet so the screenshots carry the visual weight.

## Pen input

The canvas supports pens (Apple Pencil on iPad, Windows pens, drawing tablets) through the browser's Pointer Events API; no native code is needed, and no handwriting recognition is built into the tool.

**Input handling**

- Read `pointerType` on every pointer event. In pen mode, `pen` draws, `touch` pans and zooms with one or two fingers, and `mouse` behaves as today.
- Pen mode turns on automatically at the first `pen` event and can be toggled in the top bar.
- Palm rejection: while a pen stroke is active, ignore all `touch` pointers.
- Render strokes with perfect-freehand (MIT), using `pressure` for width. Store raw points and pressure; render at display time.
- Set `touch-action: none` on the drawing surface so the browser does not scroll or zoom during strokes.

**Gesture recognition**

In pen mode, the user does not pick a tool for each mark. When a stroke ends, deterministic geometric heuristics classify it, and a chip lets the user change the result with one tap.

| Stroke | Classified as | Heuristic |
| --- | --- | --- |
| Path whose end returns within 15% of its bounding-box diagonal to its start | Circle | Closed-loop test |
| Mostly straight path ending in a short hook or two short head strokes within 400 ms | Arrow | Straightness + head detection |
| Two crossing strokes or a dense zig-zag over one element | Remove | Crossing count over resolved element |
| Several short strokes in a compact area within 1.5 s of each other | Handwriting | Stroke grouping by time and distance |
| Anything else | Freehand | Fallback |

Handwriting placed next to a circle, arrow or remove within 48 px becomes that annotation's comment, following the Resolver's attachment rule. Thresholds live in the same config object as the resolver thresholds.

**Ink data**

```ts
type InkData = {
  strokes: { points: [number, number, number][] }[];   // x, y, pressure
  pointerType: "pen" | "mouse" | "touch";
  handwriting: boolean;
};
```

**Handwritten text**

- Handwriting stays ink. The compiler crops each handwritten group into `rounds/<n>/ink/<annotationId>.png` with 16 px padding, on a transparent background.
- The instruction references the crop instead of text, for example: "\[R3-2\] Handwritten note on the Pay now button (id: payButton), see ink/a17.png". The coding agent reads the image with its vision capability; the target element stays exact.
- `review.md` embeds the crops as images, so agents that read Markdown images get them inline.
- Typed comments remain available: the comment field is a normal text input, so iPadOS Scribble converts pencil handwriting to text there with no extra work.

**Using an iPad with a computer**

Capture runs on the computer; review may happen on an iPad or tablet on the same network. `review open --lan` enables this, as described under CLI and server.

## Resolver

The resolver is a pure function in `packages/core` that turns an annotation plus an element tree into a `Resolution`; it is the part of the product that most deserves polish and tests.

```ts
resolve(annotation: Annotation, tree: UIElement, allTrees?: Map<string, UIElement>): Resolution
```

**Rules per annotation kind**

- **Comment:** attach to `attachedTo` if set. Otherwise pick the deepest element containing the pin point. If the pin sits within 24 px of another annotation's geometry, attach to that annotation instead.
- **Circle and freehand loop:** close the path into a polygon. For every element, compute coverage = area of element inside polygon ÷ element area. Candidates have coverage ≥ 0.6. Pick the smallest container whose own coverage ≥ 0.6 and whose area is ≥ 70% covered by candidates; otherwise return all top-level candidates. If nothing qualifies, return `region` with the polygon's bounding box.
- **Arrow:** resolve the start point to the deepest element under it. Resolve the end point the same way; if it lands on empty space (only the root or a full-screen container), return a `region` destination. If the end lands on another tile, set the destination on that screen.
- **Rectangle:** resolve like a circle. Low coverage means "add something here", so return `region` when no element has coverage ≥ 0.6.
- **Remove:** the deepest element under the click; the human can walk up to the parent with the target picker.
- **Freehand (open path):** resolve by the elements the path crosses, ranked by path length inside each; treat as a note on the top one.
- **Rule:** targets are set explicitly by selection; no geometry resolution.

**Tie-breaking.** Prefer elements with real ids over generated ids, then smaller area. Ignore elements larger than 90% of the screen unless nothing else qualifies.

**Human override always wins.** Once `confirmedByUser` is true, re-running the resolver must not change the resolution.

**Thresholds are configuration.** Keep 0.6, 70%, 24 px and 90% in one config object so they can be tuned against fixtures.

## Compiler

The compiler is a deterministic, template-based pure function that turns resolved annotations into `review.json`, `review.md`, and new entries in `rules.md`.

**Grouping.** A comment attached to a circle, arrow, rectangle or remove merges into that annotation's instruction. Instructions are grouped per screen in manifest order, and numbered `R<round>-<n>` across the round.

**Target phrasing.** A target is written as `<label> <type> (id: <id>)`, adding `in <component>, <file>` when `source` exists. Generated ids are written as a description plus bounds, since the agent cannot search for them.

| Annotation | Action | Template |
| --- | --- | --- |
| Remove | remove | Remove {target}. {comment} |
| Circle or comment on element | change | {target}: {comment} |
| Arrow to element | move or relate | Move {target} next to {destination}. {comment} |
| Arrow to region | move | Move {target} to the area at {region}. {comment} |
| Rectangle on region | add | Add {comment} in the empty area at {region}, between {neighbours}. |
| Arrow between tiles | relate | Flow: {screen A} leads to {screen B} via {target}. {comment} |
| Freehand | note | Note on {target}: {comment} |
| Unresolved | any | \[UNRESOLVED\] {comment} (see annotated screenshot, marker {n}); ask the user. |

When a circle or rectangle has no comment, the compiler writes "Review {target}; see marker {n}" and flags the instruction as needing text, so the canvas can warn before sending.

**Rules.** Each rule annotation is appended to `rules.md` with a date, the rule text, and the element examples it came from. It is not added to the round's instructions, but the round's `review.md` links to new rules at the top.

**review.md format**

```markdown
# Design review, round 3

New rules added to rules.md: 1. Unresolved: 1.

## Checkout (checkout-default)
Screenshot: screens/checkout-default.annotated.png

1. [R3-1] Remove the "Apply promo code" banner (id: promoBanner).
2. [R3-2] "Pay now" button (id: payButton): too dominant, make it secondary to the order summary.
3. [R3-3] Move the shipping form (id: shippingForm) below the order summary (id: orderSummary).
4. [R3-4] Add a secure-payment badge in the empty area above "Pay now" (x 32, y 612, 326 × 48).
```

**Annotated screenshots.** The compiler renders each screen's annotations onto a copy of its PNG, with numbered markers matching the instruction numbers, using the same SVG layer as the canvas rendered server-side (resvg or Playwright).

## CLI and server

The CLI is the single entry point, published to npm and run with `npx`; it starts a local server that serves the canvas and reads and writes `.review/`.

| Command | What it does |
| --- | --- |
| `review init` | Creates `.review/` with an example `screens.json`, adds the agent section to `AGENTS.md` |
| `review doctor` | Checks platform tools and devices, prints fixes |
| `review capture` | Creates a new round, runs every screen's flow, captures screenshot and tree |
| `review open` | Starts the server and opens the canvas on the latest round |
| `review` | Shortcut for `capture` then `open` |
| `review status` | Prints the latest round's state and instruction counts |
| `review mcp` | Starts the MCP server over stdio (post-MVP) |

Flags: `--platform`, `--device`, `--screens <ids>` to capture a subset, `--port` (default 4382), `--no-open`.

**Server**

- Hono on Node 20+, by default bound to `127.0.0.1` only.
- REST: `GET /api/rounds`, `GET /api/rounds/:n`, `GET /api/rounds/:n/screens/:id` (capture + tree), `PUT /api/rounds/:n/annotations`, `POST /api/rounds/:n/resolve`, `POST /api/rounds/:n/send`.
- Static files: canvas build and screenshot PNGs.
- WebSocket `/ws`: pushes `round-created`, `capture-progress` and `status-changed` events, so an open canvas updates when the agent triggers a new capture.
- A file watcher on `.review/latest/status.json` detects when the agent marks a round `applied`.

**LAN mode.** By default the server listens only on `127.0.0.1`. With `--lan`, it also listens on the computer's local network address and prints a URL plus a QR code containing a one-time pairing token. The first device to open that URL gets a session cookie; every other request without a valid session is refused. The token expires after 10 minutes or first use, and LAN mode ends when the server stops. The canvas UI must be fully usable on a 10-inch tablet screen in both orientations.

**Send.** `POST /send` runs the resolver on unconfirmed annotations, compiles, writes `review.md`, `review.json` and annotated PNGs, appends rules, and sets status to `sent`. The canvas then shows a copyable one-line prompt, for example "Implement .review/latest/review.md".

**Packaging.** One npm package with the built canvas bundled as static assets. No global install required; no database; no network access needed except by the capture tools themselves.

## Milestones and acceptance criteria

Build in six milestones; each ends in something usable, and the MVP is milestones 1 to 5. Do not start a milestone before the previous one's criteria pass.

**M1: Schemas and one manual capture**

- [ ] Monorepo set up with pnpm, TypeScript strict, Vitest, ESLint, Prettier.
- [ ] All schemas in `core` with Zod, exported types and generated JSON Schema.
- [ ] Parser for the first platform's hierarchy output, tested against 3 checked-in fixtures.
- [ ] A script turns one screenshot plus hierarchy dump into a valid `ScreenCapture`.

**M2: Canvas on static data**

- [ ] Canvas loads a fixture round and shows tiles grouped by `group`, with pan and zoom.
- [ ] Focus view with element hover outlines and the `Alt` cycle.
- [ ] All 8 tools draw, select, edit and delete; undo and redo work.
- [ ] Annotations persist to `annotations.json` and reload.

**M3: Resolver and compiler**

- [ ] Resolver implements every rule in the Resolver section, with tests per rule.
- [ ] Resolution chips and target picker in the canvas; overrides survive re-resolve.
- [ ] Compiler produces `review.md`, `review.json`, annotated PNGs and rules entries.
- [ ] Golden-file tests: 5 fixture rounds compile to checked-in expected output.

**M4: Automatic capture**

- [ ] `init`, `doctor`, `capture`, `open`, `status` work for the first platform.
- [ ] Capture runs Maestro flows per screen and writes a complete round folder.
- [ ] A failing screen is reported and skipped; the rest of the round still captures.

**M5: Closed loop (MVP done)**

- [ ] `init` writes the agent section; an agent given only that section finds and applies a review.
- [ ] WebSocket updates the canvas when a new round is captured.
- [ ] End-to-end test on a sample app: capture, annotate 5 changes, send, agent applies, recapture shows them.
- [ ] Field test: on a real app, at least 4 of 5 instructions are applied correctly without follow-up questions.

**M6: After the MVP, in order of priority**

- [ ] Rounds view: before and after per screen, accept or rework per instruction; rework carries into the next round.
- [ ] Pen input: pointer handling, gesture recognition, handwriting crops, and `--lan` pairing for tablets.
- [ ] Second mobile platform adapter, then the web adapter.
- [ ] Flow arrows between tiles rendered as a screen map.
- [ ] React Native and Flutter source mapping.
- [ ] MCP server with `request_review`, `get_feedback` (non-blocking, returns waiting or done) and `list_rounds`.
- [ ] Optional Tauri desktop wrapper.

## Testing and open questions

Most bugs will be in parsing and resolution, so test those hardest with real captured fixtures; the canvas needs fewer, focused tests.

**Testing**

- **Unit (Vitest):** hierarchy parsers, normalization, id stability, every resolver rule, compiler templates.
- **Golden files:** fixture rounds in `fixtures/` with expected `review.md` and `review.json`; any output change must update the golden file in the same commit.
- **Canvas (Playwright component tests):** tool gestures, resolution chips, target picker, undo and redo, autosave.
- **End-to-end:** a small sample app in `examples/` with 4 screens and Maestro flows, used in CI where a simulator is available and run manually otherwise.

**Open questions**

- [ ] Which platform does the first capture adapter target: iOS simulator or Android emulator?
- [ ] Product name, CLI command and folder name, replacing the `review` and `.review/` placeholders.
- [ ] Should rules live only in `rules.md`, or also be proposed as edits to the project's own design tokens or theme files?
- [ ] Is the target user mainly a designer using the tool alone, or a designer and developer pair? This affects whether comments need authors.
- [ ] License: open source (MIT) or commercial.
