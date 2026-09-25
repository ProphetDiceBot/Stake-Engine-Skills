Slot Spine authoring and integration

This skill is the single owner of Spine 4.2 authoring, rigging, animation, atlas preparation,
validation, and runtime integration for slot-game symbols, characters, heroes, and FX.

Do not duplicate Spine pipelines or introduce alternate authoring/runtime dependencies.
Use the versions, loaders, build scripts, texture pipeline, and runtime already installed by
the target game.

slot-gen is upstream artwork generation/extraction only. Once usable artwork exists, this
skill owns its placement, rigging, animation, Spine export, validation, and integration.

1. Mandatory preflight

Before modifying or generating any Spine asset:

Read the game's memory/index documentation.
Read package manifests and lockfiles.
Identify the installed:
Spine version
Pixi version
Urso version
loader/runtime version
texture-builder version
Inspect the game's existing Spine loader and asset-loading conventions.
Identify the project's canonical build, test, and preview commands.
Read:
references/spine-pixi-v8-4.2.md
references/rigging-and-animation.md
Inspect the relevant existing assets before creating a new skeleton or atlas.
Install this repository into the working Python environment:
python -m pip install -e .

The shared workspace interpreter is:

/Users/maksim/MorningCat/.local/skills-venv/bin/python

Do not assume that interpreter is the game's runtime. The game's own package/runtime configuration
is authoritative for runtime validation.

Workspace rules

Keep these categories separate:

accepted source masters
maintained Spine source kits
generated exports
temporary candidates: .tmp_<task>/
runtime preview output

Never promote a candidate to an accepted master merely because it parses successfully.

Preserve third-party terms and attribution in:

THIRD_PARTY_NOTICES.md
licenses/

Do not silently replace, vendor, or upgrade project dependencies.

2. Decide what actually needs to move

Before rigging, determine the smallest useful set of independent layers.

Separate a layer only when it needs independent:

translation
rotation
scale
deformation
tint
alpha
blend mode
draw order
looping motion
attachment replacement

Prefer a reusable rig over frame swapping.

Use video/frame swaps only when:

the motion cannot reasonably be represented by a reusable Spine rig, and
the memory cost is explicitly budgeted and accepted.

Do not create bones or slots merely because the source artwork contains separate files.

3. Coordinate and layout workflow

Use the smallest applicable pipeline stage.

Need	Authoritative tool
Canvas coordinates → pivots	scripts/px_to_skel.py
Position existing parts against reference	scripts/position_parts.py
Initial humanoid layout / inspection	scripts/canonical_layout.py
Composite layout inspection	scripts/compose_layout.py
Layout → skeleton configuration	scripts/build_skeleton_v2.py or scripts/layout_to_spine_config.py
Configuration → Spine JSON	scripts/build_spine_json.py
Standalone source atlas	scripts/make_atlas.py
Runtime parsing / sampled motion	scripts/validate-spine.mjs
Installed-runtime preview	scripts/build-runtime-preview.mjs
Atlas memory / attachment keys	scripts/check-atlas-budget.mjs

Run Python commands with --help before using unfamiliar options.

Node validators must receive explicit paths where supported:

--project-root
--spine
--atlas-root
--out

The runtime preview requires a new or empty output directory.

4. Canonical coordinate and pivot rules

Use one canonical canvas throughout the asset.

Do not independently reposition artwork after skeleton coordinates have been established.

Pivots must represent the actual semantic anchor:

shoulder → shoulder joint
elbow → elbow joint
wrist → wrist joint
hip → hip joint
foot → contact point
prop → grip/contact point
FX → intended visual origin
symbol → intended visual center or attachment point

Do not use image corners as pivots unless the corner is intentionally the attachment anchor.

When translating artwork between coordinate systems, use the project's coordinate-conversion tools.
Do not hand-compensate offsets repeatedly.

5. Layer and slot construction

Slot order is back-to-front.

Establish draw order from the final composite, not from source-file naming.

Before exporting:

verify every required attachment exists
verify attachment keys match runtime expectations
verify slot ordering
verify foreground/background coverage
verify frame, halo, shadow, and FX coverage
remove baked elements that are being replaced by animated elements
Ghost prevention

If an element is animated independently, its baked/static version must not remain underneath it.

Typical failure:

baked arm + animated arm

Correct:

animated arm only

unless the baked layer is intentionally part of the final composite.

6. Rigging rules

Build the minimum hierarchy required by the motion.

Prefer:

root
├── body
├── head
├── upper_arm
│   └── lower_arm
│       └── hand
└── upper_leg
    └── lower_leg
        └── foot

over excessive per-layer bones.

Use meshes only when deformation materially improves the result.

Use rigid attachments when rotation/translation is sufficient.

Use continuous FX meshes/layers when the effect benefits from:

deformation
looping motion
procedural-looking movement
continuous alpha
continuous rotation
reusable animation

Do not introduce constraints simply because Spine supports them. A constraint must solve a
specific registration or motion problem.

7. Animation construction

Typical animation set:

idle
walk
run
win

Only create animations required by the game.

Every animation must define:

duration
keyed channels
loop behavior
setup pose relationship
intended start pose
intended end pose
required attachment visibility
required draw-order state
tint/alpha/blend behavior where applicable
Looping animations

A looping animation must have a deliberate join.

Do not rely on the player interpolating from an arbitrary final pose back to the first pose.

For a loop:

sample the start pose
sample the end pose
compare the relevant bones/attachments/channels
correct the endpoint where necessary
validate the join against neighboring samples

Use:

scripts/validate-spine.mjs --loop <animation>

when loop validation is applicable.

Spine curve rules

Rotate timelines use value.

Setup bones use rotation.

Bezier points are absolute time/value coordinates per channel.

The builder may convert preset easing into Spine curves.

Do not convert already-exported custom Spine curves a second time.

8. FX and blend behavior

Treat alpha, tint, and blend mode as part of the asset's visual contract.

Validate:

alpha transitions
tint channels
additive/screen/normal behavior where applicable
foreground/background interaction
clipping or masking
edge behavior
overlap with symbol/frame/halo elements

A mathematically valid Spine file is not sufficient if the runtime appearance is wrong.

9. Atlas construction

For standalone source atlases use:

scripts/make_atlas.py

For runtime texture validation use:

scripts/check-atlas-budget.mjs

Provide a budget JSON containing:

{
  "maxDecodedBytes": 0,
  "maxPages": 0,
  "maxEdge": 0
}

Validate every configured PNG/WebP.

Reject:

missing attachment keys
missing textures
unexpected atlas pages
oversized pages
oversized decoded memory
clipped canonical layers
accidental duplicate textures
incorrect attachment paths

RGBA decoded-memory estimates exclude mipmaps and driver overhead. Treat the configured budget
as a minimum accounting requirement, not a complete GPU-memory guarantee.

10. Runtime integration

Integration must use the game's existing loader.

noAtlas

Resolve attachment paths through the game's shared texture keys.

Explicit Spine atlas

Preserve the loader's existing atlas ordering and naming conventions.

For multipack assets, load the multipack root _0 once according to the project's loader contract.

Do not:

add a second loader
fake runtime startup
create a parallel Pixi/Spine version
bypass the game's texture builder
manually patch runtime state to make a broken asset appear correct
accept a canvas-only render as proof of integration

Build using the project's actual build scripts.

When requested, use:

zephyr-launcher-session

for desktop/mobile runtime evidence.

11. Validation gates

Validation happens in four increasingly authoritative stages.

Gate A — Structural

Confirm:

JSON parses
atlas parses
attachment keys exist
referenced textures exist
skeleton names are correct
animations exist
animation durations are finite
no required fields are malformed

Structural success does not imply visual correctness.

Gate B — Motion

Using:

scripts/validate-spine.mjs

verify:

setup pose
animation sampling
bone transforms
attachment state
loop boundaries
fixed channels
mesh geometry
relevant constraints
tint/alpha where supported
Gate C — Installed-runtime preview

Build with:

scripts/build-runtime-preview.mjs

Then inspect:

setup pose
several points in each animation
first loop
loop join
mesh deformation
alpha
tint
blend mode
foreground/background ordering
frame/halo coverage

preview_spine.py is only a limited region/contact-sheet preview.

It is not authoritative for:

mesh deformation
constraints
tint
advanced blend modes
full installed-runtime behavior

Legacy:

generate_spine_player.py

is a convenience CDN player only. It is never the installed-runtime acceptance gate.

Gate D — Actual game runtime

When integration is requested, validate through the installed game/runtime.

This is the final visual and integration authority.

A passing standalone preview does not override a failing installed-runtime result.

12. Evidence requirements

For accepted work, retain evidence showing:

skeleton/setup pose
each requested animation
at least one loop boundary for looping animations
atlas/texture validation
runtime preview
installed-runtime evidence when integration was requested
any known limitations

Evidence must demonstrate actual behavior, not merely successful file generation.

Do not claim:

"validated"

when only JSON parsing occurred.

Use precise status language:

parsed
structurally validated
motion validated
preview validated
installed-runtime validated
accepted
13. Failure handling

When a gate fails:

identify the first failing layer
preserve the failing candidate
inspect the relevant source/config/runtime evidence
fix the smallest responsible component
rerun the failed gate
rerun downstream gates affected by the change

Do not repeatedly regenerate the entire asset when the failure is isolated.

Examples:

wrong pivot
→ fix pivot
→ rebuild affected skeleton data
→ rerun motion/runtime validation
missing atlas key
→ fix atlas/config
→ rerun atlas validation
→ rerun runtime validation
incorrect blend mode
→ fix runtime/Spine property
→ rerun preview
→ rerun installed-runtime validation

Never hide failures by modifying the validator or weakening acceptance criteria.

14. Source-of-truth hierarchy

When sources disagree, use this authority order:

installed game runtime
        ↓
game loader / runtime contract
        ↓
project build + texture configuration
        ↓
maintained Spine source
        ↓
generated preview
        ↓
temporary candidate

A lower-level artifact cannot override observed behavior at a higher-authority layer.

15. Completion criteria

A Spine task is complete only when all requested stages have passed.

Authoring-only task

Must have:

maintained source kit
one canonical skeleton
required animations
required atlas/config
structural validation
motion validation
visual preview
documented limitations
Integration task

Must additionally have:

project-native build
existing loader integration
installed-runtime validation
runtime evidence
texture-memory validation

Do not mark an asset accepted solely because:

Spine opens it
JSON parses
an atlas exists
a standalone preview renders
a CDN player renders it
a screenshot looks plausible
16. Cleanup and preservation

Preserve recovery material until acceptance.

Cleanup is scoped to the current task.

Never perform broad automatic deletion of:

source masters
maintained skeletons
accepted atlases
runtime evidence
recovery candidates
third-party notices/licenses

Temporary .tmp_<task>/ material may be removed only after acceptance and only when it is no
longer required for recovery or audit.

17. Final deliverable

Finish with:

source/
  maintained Spine source kit

skeleton/
  one maintained skeleton/config

atlas/
  accepted atlas assets

animations/
  requested animation data

validation/
  structural + motion + atlas evidence

runtime/
  installed-runtime evidence when requested

limitations.md

The final report must state:

what was authored
what was reused
what was changed
which validation gates passed
which runtime was actually tested
atlas/memory status
known limitations
any intentionally unvalidated behavior

Do not report assumptions as validation.
Do not report preview success as runtime success.
Do not report structural validity as visual acceptance.
