---
layout: doc
title: Audio Face Animator documentation
description: Complete documentation for Audio Face Animator — the Unity plugin that generates ARKit facial animation from audio on your own machine, using the NVIDIA Audio2Face-3D SDK. Setup, character mapping, emotions, eyes and blinking, the scripting API and troubleshooting.
parent: Audio Face Animator
parent_url: /audio-face-animator/
updated: 2026-09-02
nav_blurb: generate lip-sync and facial animation for a Unity character from an audio file, with NVIDIA Audio2Face
---

**Audio Face Animator** generates facial animation from audio using the **NVIDIA
Audio2Face-3D** model, driving any character with **ARKit-compatible
blendshapes**. Everything runs on your own machine — no cloud service, no
account, no API key and no per-request cost, with the model, CUDA and TensorRT
all inside the package.

Since version 1.1 a performance can also carry an **emotion**, the eyes can be
driven from the rig's own eye bones, and the character **blinks** on its own.
Individual ARKit shapes can drive **several blendshapes or a bone**, so rigs
whose jaw is a bone are supported too.

## Requirements

<div class="table-scroll" markdown="1">

| Requirement | Minimum |
|---|---|
| **Platform** | Windows 10 / 11, 64-bit |
| **GPU** | NVIDIA, compute capability 7.5 or newer — GeForce RTX 20-series or newer, or the equivalent Quadro / RTX A-series / data-center card |
| **Driver** | Recent enough for CUDA 12.9; version 576 or newer recommended |
| **VRAM** | 6 GB or more free while generating |
| **Unity** | 6000.0 or newer |
| **Disk** | About 3 GB, and roughly the same again while the plugin syncs its model files |

</div>

Built-in, URP and HDRP are all fine — the plugin drives blendshape weights and
transforms, and never touches materials or shaders.

## Versions Comparison: Lite vs Full
{: #editions}

![Feature comparison: both versions generate facial animation from audio, process fully offline and work in the Unity Editor. Only the full version supports standalone builds and unlimited audio length; Lite is capped at 5 seconds of audio and animation](/audio-face-animator/docs/img/face-animator-versions-comparison.webp)

The Lite version has two limitations:

- **Audio length is limited at five seconds.**
- **It works in the Unity Editor only** — no standalone builds.

Everything else — mapping, emotions, eyes, blinking, AnimationClip export — is
the same in both.

If you own the full version and still see a five-second cap or an orange
**⚠️ LITE VERSION** banner on the Face Animator, the model files did not arrive
intact; see [Troubleshooting](#troubleshooting).

## Configuration

Both steps are prompted automatically when the package is imported, and both are
one-time. Use the menu items below to repeat them later.

The package adds three menu items, all under **Tools → AudioFaceAnimator**:
*Sync StreamingAssets*, *Setup TensorRT Engine* and *Documentation*, which opens
this page.

### 1. Sync StreamingAssets

**Tools → AudioFaceAnimator → Sync StreamingAssets**, then **Copy All Files**. Run
it again after every plugin update.

![The StreamingAssets Sync window, listing model files missing from Assets/StreamingAssets/DevEloop/AudioFaceAnimator](/audio-face-animator/docs/img/streamingassets-sync.png)

This copies the model files to `Assets/StreamingAssets/DevEloop/AudioFaceAnimator`,
which is where Unity looks for them at runtime. The window compares file
*content*, not timestamps, so an updated model file shows up as a mismatch and a
damaged one is repaired by copying again. *No Differences* means there is nothing
to do.

### 2. Setup TensorRT engine

**Tools → AudioFaceAnimator → Setup TensorRT Engine**, then **Build Engine for
This GPU**. About a minute, once per machine.

![The TensorRT Engine window: the detected GPU, a warning that no engine exists yet, and the Build Engine for This GPU button](/audio-face-animator/docs/img/tensorrt-engine-setup.png)

The engine is compiled from the `network.onnx` synced in step 1, because a
compiled engine only loads on the GPU architecture it was built for. Leave *Build
the full batch range declared by the model* off — it doubles the engine size and
reserves about 2 GB more GPU memory for a capability the plugin never uses.

The result is cached at

```text
%USERPROFILE%\AppData\LocalLow\<company>\<product>\DevEloop\AudioFaceAnimator\engine\<gpu-signature>\
```

and reused on every later run. The cache key is your GPU's signature plus a
fingerprint of `network.onnx`, so the engine survives plugin updates but is
rebuilt — correctly — after a GPU or driver change, or when the model itself
changes. **Clear Built Engine Cache** in the same window forces a rebuild.

In a standalone build the first generation call does this automatically, on each
player's machine.

## Preparing your character

Two components: a **Face Blends Mapper** on each face renderer, and one
**Face Animator** on the character that drives them.

### 1. Face Blends Mapper

Add it and assign your face `SkinnedMeshRenderer` to **Target Renderer**.

![The Face Blends Mapper component with no Target Renderer assigned, showing the notice "Mappings not initialized. Assign a SkinnedMeshRenderer with blendshapes."](/audio-face-animator/docs/img/face-blends-mapper-empty.png)

It immediately creates all 52 ARKit rows and matches them against the mesh's
blendshapes — ignoring case, spaces and underscores, and accepting prefixed names
like `CC_Base_Body.jawOpen`. The console reports the result, naming every row it
could not match.

**Add one mapper per face renderer** (head, teeth, tongue and eyes are often
separate). The three shipped prefabs carry five each.

![The Face Blends Mapper after auto-mapping: the Preset row with Save, Load and Save As, both Auto-Map buttons, the Filter box, and the first ARKit rows matched to mesh blendshapes with their max-weight sliders. Eye Blink Left carries a +1 badge for its second blendshape target, Jaw Open a 1b badge for a bone target](/audio-face-animator/docs/img/face-blends-mapper-mapped.png)

Matching is deliberately generous, so read down the list once. A **Filter** box
narrows the 52 rows while you work, and each row expands to show everything it
drives:

- **Blendshapes** — one or more mesh blendshapes, each with its own **max weight**
  (0–100, default 100). Max weight is both scale and ceiling: 50 halves the
  shape's travel. This is the tuning control — lower `jawOpen` if the mouth opens
  too far. Use **Add blendshape target** for a rig that splits one ARKit shape
  across several shapes, or needs a corrective; the collapsed row then shows a
  `+N` badge.
- **Bones** — see below.
- **Preview** — a 0–1 slider that drives the row live in the Scene view, without
  generating anything. The quickest proof that a row is wired correctly.
  Collapsing the row, or selecting another object, returns the face to rest.

![One ARKit row expanded: a Blendshapes list with two targets, eyeBlinkLeft at max weight 100 and eyeSquintLeft at 60, an Add blendshape target button, an empty Bones section, and the row's own Preview slider](/audio-face-animator/docs/img/face-blends-mapper-row-expanded.png)

**Auto-Map (fill empty)** touches only unmapped rows; **Auto-Map (overwrite all)**
redoes everything, discarding manual edits. Several rows may point at the same
mesh blendshape — their contributions are summed and clamped, so nothing is lost.

`jawOpen`, `mouthClose`, `mouthFunnel` and `mouthPucker` are the rows speech
cannot do without. Check those first.

#### Driving a bone instead of a blendshape

For a rig whose jaw, lips or brows are bones, expand the row and press **Add bone
target**.

![The Bones block of the Jaw Open row: a Bone field holding a transform, the captured rest pose with a Re-capture rest button, a Rotation (deg) delta of 14 degrees on X, a zero Position offset, an Influence slider at 1, and a Capture delta from current pose button](/audio-face-animator/docs/img/face-blends-mapper-bone-target.png)

Assign the **Bone** and its current local pose is captured as the rest pose
immediately — so do this with the rig neutral, or press **Re-capture rest**
afterwards. Then enter the pose the expression should reach, either by typing
into **Rotation (deg) delta** and **Position offset**, or by posing the bone in
the Scene view and pressing **Capture delta from current pose**. **Influence**
scales the whole target. Verify with the row's **Preview** slider: 0 is rest, 1
is the captured pose.

Three rules that will otherwise bite:

- **A bone may appear in only one mapper.** Several rows *within* one mapper may
  share a bone and accumulate; two mappers claiming the same bone is an error.
- **Bones must be descendants of the Face Animator's transform** or their curves
  are skipped when baking.
- **Eye bones belong to the Eyes Rotation Mapper**, not here. A bone claimed by
  both is an error.

#### Reusing a mapping

Press **Save As…** in the mapper's **Preset** row to write a `FaceBlendsPreset`
asset, then drop it into the **Preset** field on the next character and press
**Load**. **Save** overwrites the selected preset.

A preset stores blendshape and bone **names**, not object references. Blendshape
names must match the new mesh exactly — loading a preset does not re-run the
fuzzy auto-map — and bones are resolved by searching the transforms under the
mapper's own GameObject, so bone targets only transfer when the mapper sits
inside the rig. The console reports exactly what was applied and what it could
not find.

### 2. Face Animator

Add it to the character root and drag every mapper into **Blendshapes
Configuration → Blends Mappers**. Any renderer whose mapper is not listed will
not move.

![The Face Animator component with its Blends Mappers list expanded, showing five Face Blends Mapper references](/audio-face-animator/docs/img/face-animator-blends-mappers.png)

## Generate animation in Editor

![The Face Animator component in the Inspector: Model Type, Blendshapes Configuration, Model Configuration, Eyes, Blinking, and Animation Playback with an Audio Clip and an Emotion Preset assigned. Outside Play mode a notice reads "Enter Play Mode to generate and play animation" and the Generate, Play and Stop buttons are disabled](/audio-face-animator/docs/img/face-animator-inspector.png)

**Model Type** — which Audio2Face-3D model generates the animation: **Claire**,
**James** or **Mark**. Each was trained on a different performer and produces a
different speaking style. This is a *performance style*, not a character: it has
nothing to do with what your character looks like, and Claire will drive a male
character perfectly well. Try all three on the same audio.

**Audio Clip**, under *Animation Playback* — the speech to animate. Any length,
any sample rate, mono or stereo; the plugin resamples to 16 kHz mono itself. No
other preparation is needed, though a clip whose import **Load Type** prevents
`AudioClip.GetData` from returning samples cannot be read.

Then **enter Play mode** — outside it the inspector says so and **Generate** is
disabled — press **Generate**, and press **Play** to preview the result. **Stop**
ends it early and resets the face to neutral. Generation runs on a background
thread, so the Editor stays usable.

On the very first generation the plugin may pause to compile a TensorRT engine,
logging progress to the console. Changing the AudioClip discards the generated
animation; changing the Model Type does not, so regenerate to hear it.

### Saving an AnimationClip

![The same component in Play mode after a successful generation: Generate and Play are enabled, Stop is disabled, and a Create AnimationClip button has appeared below them. Unity tints its interface slightly darker in Play mode](/audio-face-animator/docs/img/face-animator-generate-playmode.png)

Once animation exists, a **Create AnimationClip** button appears. It writes a
standard Unity `AnimationClip`, which must be saved **inside your project's
`Assets` folder**. The button is only there while generated data exists, which
means **before you leave Play mode** — leaving discards it.

What lands in the clip: one curve per mapped blendshape per renderer, rotation
curves for every driven bone (plus position curves only for bones some row
actually offsets), the procedural blink, and the eye rotations if an Eyes
Rotation Mapper is assigned. Tangents are smoothed and the clip is a normal
non-legacy clip.

It has no dependency on the plugin, the SDK or an NVIDIA GPU, so it plays from an
Animator or Timeline on any platform Unity builds for. Three things to know:

- **Curve paths are relative to the GameObject holding the Face Animator**, so
  moving that component to a different level of the hierarchy invalidates the
  clip. Put it on the character root and keep it there.
- **Weights are baked in** when the curves are written. Changing a mapper weight
  afterwards means generating and saving again.
- **Set a non-zero blink seed before baking a set of clips**, or every bake
  blinks at different moments and re-baking one line makes it inconsistent with
  the rest.

### The sample scene

![The sample scene running: a head on a dark background, with a model dropdown set to James, Generate, Play and Stop buttons down the left side, and a Browse button in the top right](/audio-face-animator/docs/img/sample-scene.png)

`Assets/DevEloop/AudioFaceAnimator/Scenes/AudioFaceAnimator.unity` has all three
characters set up, a model selector, generation controls and a file browser for
loading a WAV from disk. It is the quickest way to confirm the plugin works on
your machine, and a working reference for wiring the API to UI.

## Emotions

Without a preset the model is driven with a neutral emotion, which reads as a
competent but flat performance. An **Emotion Preset** colours it.

Press **Create…** next to the **Emotions** field on the Face Animator: it saves a
new preset asset and assigns it in one step. An existing preset can be dropped
into the same field, and presets can also be made from *Assets → Create →
DevEloop → Audio Face Animator → Emotion Preset*.

![The Emotion Preset inspector: three segments, amazement, cheekiness and joy, each with a share of 1 and an intensity of 1; a Crossfade of 0.25 seconds; a Preview Clip length of 5 seconds; and a coloured bar underneath giving each emotion 33.3 percent, or 1.67 seconds, of the clip](/audio-face-animator/docs/img/emotion-preset-inspector.png)

A preset is an ordered list of segments, read left to right across the clip:

<div class="table-scroll" markdown="1">

| Emotion | Share | Intensity |
|---|---|---|
| fear | 0.1 | 1 |
| anger | 0.3 | 1 |
| fear | 0.4 | 0.6 |
| joy | 0.2 | 1 |

</div>

**Share** is how much of the clip the segment occupies, not how strong it is.
Shares are normalised, so they do not have to add up to anything and the same
preset fits a clip of any length — the example above gives 10%, 30%, 40% and 20%
of whatever clip you generate from, and writing it as 1, 3, 4, 2 means exactly the
same thing. *Normalize Shares* rewrites them as fractions of one, which changes
the numbers and not the result.

**Intensity** is the value handed to the model, 0 to 1. Start around 0.5 and
raise it until the performance reads the way you want; 1 is the strongest the
model accepts, and anything above is clamped.

**Crossfade** is how long one segment takes to blend into the next, in seconds
(0–2, default 0.25). It is applied symmetrically around each boundary and clamped
per boundary, so a segment is never swallowed by its own transitions; a segment
too short to hold anything becomes a brief peak instead. Raise it if transitions
pop.

The coloured bar under the list previews the result — each slice shows its
emotion, its percentage and its duration against **Preview Clip (s)**, which only
scales the seconds shown. The preset itself is never tied to a particular clip.

The available emotions come from the model, and are listed in `network_info.json`:
**amazement**, **anger**, **cheekiness**, **disgust**, **fear**, **grief**,
**joy**, **outofbreath**, **pain** and **sadness**. A segment carries one emotion
at a time; an emotional rise and fall is built from several segments, as `fear`
is above. Four ready-made presets ship in
`Assets/DevEloop/AudioFaceAnimator/EmotionPresets/`.

Emotions are a **generation-time** input, so assign the preset before generating:

```csharp
faceAnimator.Emotions = myEmotionPreset;   // null goes back to neutral
await faceAnimator.GenerateAsync();
```

## Eyes and blinking

The model drives the face but produces almost no eyelid motion, and its eye
rotations need bones to apply them to. Both gaps are filled inside Unity.

### Blinking

Generated in Unity and mixed over the model's output, both during playback and at
bake time. It is **on by default** in the *Blinking* section of the Face
Animator, and needs `eyeBlinkLeft` and `eyeBlinkRight` mapped to real
blendshapes — without them there is no blink and no warning either. `eyeSquint*`
and `eyeWide*` are used as well when mapped, so the lower lid follows and a wide
eye does not fight a closed lid.

![The Blinking section of the Face Animator: Enabled ticked, Min and Max Interval of 2 and 6 seconds, Close, Hold and Open Duration of 0.07, 0.02 and 0.18, Amplitude 1, Squint Amount 0.2, and a Random Seed of 0](/audio-face-animator/docs/img/face-animator-blinking.png)

<div class="table-scroll" markdown="1">

| Field | Default | What it does |
|---|---|---|
| **Enabled** | on | Turn it off if your own system already blinks |
| **Min / Max Interval** | 2 / 6 s | The random pause between blinks. Narrow it to 1–2.5 s while checking that blinking works at all — with the default range a short clip may show one blink or none |
| **Close / Hold / Open Duration** | 0.07 / 0.02 / 0.18 s | Real blinks close fast and open slower. Raise both for a tired character |
| **Amplitude** | 1 | How far the lids travel; 0.6 gives half-blinks |
| **Squint Amount** | 0.2 | How much `eyeSquint` comes along |
| **Random Seed** | 0 | 0 gives different timing on every playback and every bake; any other value is repeatable |

</div>

Blink weights pass through the mapper like any other weight, so a blink shape
with **max weight** 50 blinks only halfway. Tune the amplitude or the ceiling,
not both at once.

### Eye rotation and gaze

Add an **Eyes Rotation Mapper** component and assign it to the Face Animator's
**Eyes** field. With that field empty the eyes are simply skipped, and *nothing is
logged* — it is the most common reason for "the eyes still don't move".

Point **Right Eye → Bone** and **Left Eye → Bone** at the two eye bones, with the
rig at rest; each bone's local rotation is captured as its rest pose on
assignment. Then use the component's **Preview** pitch and yaw sliders to
calibrate: the mapper rotates each bone about a **Pitch Axis** and a **Yaw Axis**
in its own local space, and rig conventions differ. If an eye rolls instead of
turning, change the axis; if it moves the right way but reversed, set that eye's
**Pitch Scale** or **Yaw Scale** to −1. Both eyes have independent axes and
scales, which is what a mirrored rig needs. **Stop preview** returns them to rest.

![The Eyes Rotation Mapper component: Right Eye and Left Eye, each with a bone, a captured rest rotation, and its own pitch and yaw axes and scales; a Response group with Rotation Strength 1, Gaze Symmetry 1, pitch and yaw limits of minus 25 to 25 degrees and Smoothing 0.03; a Look At group pointing at the Main Camera; and a Preview group with Pitch and Yaw sliders and a Stop preview button](/audio-face-animator/docs/img/eyes-rotation-mapper.png)

<div class="table-scroll" markdown="1">

| Field | Default | What it does |
|---|---|---|
| **Rotation Strength** | 1 | Scales the model's gaze for both eyes. 0 freezes it while leaving Look At working |
| **Gaze Symmetry** | 1 | Blends both eyes toward their average. 1 keeps them parallel |
| **Pitch / Yaw Limits** | ±25° | A hard clamp on where the eyes can point — useful for keeping the eyeball inside the lid opening |
| **Smoothing** | 0.03 s | Raise toward 0.08 to calm twitchy eyes, lower for snappier saccades |
| **Look At Target** | none | A transform to hold: the camera, the player's head, a marker. Its contribution is *added* to the audio-driven gaze, so the performance still reads on top of the eye contact |
| **Look At Weight** | 1 | Scales that contribution |
| **Look At Convergence** | 1 | 1 aims each eye from its own position, giving true convergence; lower it if a close target looks cross-eyed |
| **Follow Target When Idle** | off | Keeps eye contact between lines, not only during them |

</div>

The eye bones must not appear in any Face Blends Mapper, and both blink and eye
rotation are included when you press **Create AnimationClip**. One limitation
there: a **Look At target is sampled once, at bake time, at its current
position** — a target that moves at runtime cannot be baked, so either keep the
plugin's own playback or clear the target and drive eye contact yourself.

## Tuning the solver

When the animation is close but too soft, too strong or too twitchy, tick
**Model Configuration → Override Model Config**. The fields are filled from
`model_config_<Model Type>.json` in StreamingAssets and start being sent to the
engine, so generation continues unchanged until you actually move something.
Change one parameter at a time and regenerate.

![The Model Configuration section with Override Model Config ticked, so the Model Config parameters are editable: Input Strength, a Skin group of smoothing and strength sliders, an Eyelids and Lips group, an Eyes group with the gaze and saccade values, and a Reset to Model Defaults button](/audio-face-animator/docs/img/face-animator-model-config.png)

<div class="table-scroll" markdown="1">

| Parameter | Range | Default | What it does |
|---|---|---|---|
| **Input Strength** | 0–3 | 1 | How hard the audio drives the whole face. Raise for a mumbling take |
| **Lower Face Strength** | 0–2 | 1.3 | Jaw and mouth travel |
| **Upper Face Strength** | 0–2 | 1 | Brows and eyes |
| **Lower Face Smoothing** | 0–0.1 | 0.0023 | Raise to calm a twitchy mouth, at the cost of crispness |
| **Upper Face Smoothing** | 0–0.1 | 0.001 | The same for the upper face |
| **Skin Strength** | 0–2 | 1 | Overall skin deformation |
| **Face Mask Level / Softness** | 0–1 / 0.001–0.5 | 0.6 / 0.0085 | Where the model blends between upper and lower face |
| **Lip Open Offset** | −0.2–0.2 | −0.03 | Static lip separation; negative closes the mouth at rest |
| **Eyelid Open Offset** | −1–1 | 0.06 | Static eyelid opening |
| **Blink Strength** | 0–2 | 1 | The model's own eyelid motion, which is almost nothing — real blinks come from the procedural blink above |
| **Eyeballs / Saccade Strength** | 0–2 | 1 / 1 | Audio-driven gaze, and the random saccade layer on top of it |
| **Saccade Seed** | 0–4999 | 0 | Changes the random gaze pattern |

</div>

**Reset to Model Defaults** reloads the values for the current Model Type;
unticking the checkbox hands control back to the engine entirely. Values are
clamped to the ranges the engine accepts, with a warning if a hand-edited JSON
file is out of range.

Note the difference between this and a mapper's max weight: these parameters
change the whole performance, a max weight fixes one expression on one rig.

## Generate animation at runtime

Runtime generation in a standalone player is a [full-version](#editions) feature;
Lite generates in the Editor only.

```csharp
using DevEloop.AudioFaceAnimator;

public class Example : MonoBehaviour
{
    public FaceAnimator faceAnimator;
    public AudioClip clip;

    async void Speak()
    {
        faceAnimator.AudioClip = clip;
        await faceAnimator.GenerateAsync();

        if (!faceAnimator.HasAnimationData)
            return;          // generation failed; the console says why

        faceAnimator.Play();
    }
}
```

`GenerateAsync()` does not throw for the ordinary failures — a missing GPU, no
engine, an unreadable model file. It logs the reason and leaves
`HasAnimationData` false, so check that rather than catching exceptions. Use
`IsGenerating` to disable your UI while it runs, as the sample scene does, and do
not start a second generation while one is running.

<div class="table-scroll" markdown="1">

| Member | What it does |
|---|---|
| `AudioClip AudioClip { get; set; }` | The clip to animate. Assigning a different clip discards the generated animation. |
| `AudioFaceModel model` | `Claire`, `James` or `Mark`. |
| `List<FaceBlendsMapper> blendsMappers` | The mappers to drive. |
| `EmotionPreset Emotions { get; set; }` | The preset the next generation uses; `null` is neutral. |
| `EyesRotationMapper EyesMapper { get; set; }` | The eye bones to drive, or `null` for no eye rotation. |
| `ProceduralBlink Blink { get; }` | The blink settings. Only `Enabled` is public; the timings are inspector-only. |
| `bool OverrideModelConfig { get; set; }` / `ModelConfig Config { get; }` | Whether the solver parameters are sent, and the values themselves. |
| `void ResetModelConfigToDefaults()` | Reloads `model_config_<Model>.json` into `Config`. |
| `Task GenerateAsync()` | Generates on a background thread. Await it. |
| `void Play()` / `void Stop()` | Play the animation with its audio; stop and reset to neutral. |
| `AnimationClip CreateAnimationClip(string name = "")` | Builds a clip from the current animation, or `null` if nothing was generated. It returns the clip; saving the asset is the caller's job. |
| `bool IsGenerating` / `bool HasAnimationData` / `bool IsPlaying` | State, for driving your own UI. |

</div>

For audio that is not an asset in your project — a file the player picks, or one
downloaded — the plugin includes a WAV loader, which returns `null` and logs on
failure:

```csharp
AudioClip clip = await AudioUtils.LoadWavClipAsync(path);   // local path or URL
if (clip != null)
    faceAnimator.AudioClip = clip;
```

`FaceAnimator` adds an `AudioSource` in `Awake` if the object has none, so a
character instantiated at runtime needs no extra setup for playback.

### Shipping a standalone build

Run **Sync StreamingAssets** before building — the plugin reads
`Application.streamingAssetsPath`, and the copy inside the package folder is a
source, not what ships. A build carries about **2 GB**: `network.onnx` alone is
691 MB, and the CUDA and TensorRT DLLs are not trimmable.

**Keep `network.onnx` in StreamingAssets and let the first generation call build
an engine on each player's machine.** Your players' GPUs are not yours, and the
engine built on your machine will not load on most of theirs. There is no way to
build one engine covering many architectures, so this is the only approach that
works everywhere. The build takes about a minute, happens once per machine, and
is cached under `Application.persistentDataPath`, which stays writable even when
the game is installed somewhere read-only.

A minute of silence looks like a hang, so pre-warm the engine behind a loading
screen of your own rather than letting it land on the player's first line:

```csharp
EngineStatus status = EngineProvisioner.GetStatus(out string detail);

if (status == EngineStatus.Unavailable)
    return;                        // no qualifying GPU; log `detail` and fall back

if (status == EngineStatus.NeedsBuild)
    await Task.Run(() => EngineProvisioner.EnsureEngine(false, Report, out string error));

// Called on the worker thread: store the values, render them on the main thread.
// Return false to cancel a build in progress.
bool Report(string phase, float progress) { Phase = phase; Progress = progress; return true; }
```

`EnsureEngine` blocks for about a minute, so it belongs on a worker thread.
`GetStatus` doubles as the **capability check to run at startup**: a large share
of players will not have a qualifying NVIDIA GPU, and the fallback worth building
is baked `AnimationClip` assets for every line you know in advance. They play on
any GPU and any platform, with no plugin involved, which makes runtime generation
an enhancement for dynamic lines rather than a requirement.

You can additionally ship a `network.trt` in
`Assets/DevEloop/AudioFaceAnimator/StreamingAssets`, copied out of your engine
cache. Players whose GPU it fits skip the build; everyone else falls back to the
ONNX. It only helps players on the same GPU architecture as the machine that
built it, so treat it as a minor optimisation, never a substitute.

If your game only needs a fixed set of lines, you probably want none of this in
the build. Generate in the Editor, save AnimationClips, and leave the plugin's
StreamingAssets out — the clips carry no dependency on it, and local inference is
not small.

## Troubleshooting

Open the Unity console first. Every failure reports the actual reason, including
what TensorRT itself said, and failures print a specific line followed by a
generic one — the first is the useful one.

### Setup and the GPU

<div class="table-scroll" markdown="1">

| Symptom | Cause and fix |
|---|---|
| *"No usable CUDA device"*, or the native plugin could not be loaded | No NVIDIA GPU was found, or the driver is too old. Update to driver 576 or newer. On switchable-graphics laptops, make sure Unity runs on the NVIDIA GPU. |
| *"The TensorRT engine … cannot be loaded on this GPU"* | A `network.trt` you supplied was built for a different architecture. Build one via **Setup TensorRT Engine**, or remove the file and let the plugin build from `network.onnx`. |
| *"… is not a TensorRT engine (N bytes)"* | A `network.trt` did not arrive intact — usually Git LFS storing a pointer instead of the file. Delete it and rebuild. |
| *"no execution context could be created"* | Not enough free VRAM. Close other GPU-heavy applications. If you enabled the full batch range, rebuild with it off. |
| *"network.onnx is missing"*, *"trt_info.json is missing"*, *"Using built-in defaults"*, *"Using the built-in emotion names"* | Run **Tools → AudioFaceAnimator → Sync StreamingAssets**. |
| The engine build fails or is very slow | Building needs several GB of free VRAM and disk. Close other GPU applications; the console shows what TensorRT reported. |
| The Sync window opens on every Editor start | Files still differ, usually because the sync was cancelled. Press **Copy All Files** and let it finish. |
| The engine rebuilds after a driver or plugin update | Expected: the cache key is the GPU signature plus a fingerprint of `network.onnx`. |
| Animation is capped at 5 seconds, or a **⚠️ LITE VERSION** banner appears unexpectedly, or a build refuses to generate with code `-2` | All three are the same thing: the model package is incomplete. In the Lite edition they are the intended limits — [the full version is on the Asset Store](https://assetstore.unity.com/packages/slug/383624). If you own the full version, the model data did not fully arrive: run **Sync StreamingAssets**, press **Copy All Files**, and re-import the package if that does not settle it. A partial checkout is the usual cause, and it can leave everything else looking fine. |

</div>

### Mapping and playback

<div class="table-scroll" markdown="1">

| Symptom | Cause and fix |
|---|---|
| **Generate** is greyed out | It is enabled only in Play mode, and only with an Audio Clip assigned. |
| **Create AnimationClip** is missing | There is no generated data — usually because Play mode was left. Generate again and bake before leaving. |
| Generation succeeds but nothing moves | The character is not mapped. Check the mappers are listed in **Blends Mappers**, and drag a row's **Preview** slider to 1 to test the routing on its own. |
| *"Mapper X has nothing to drive, skipping"* | Every blendshape name in that mapper is missing from the mesh, and it has no bones. Re-run auto-map, or check you assigned the right renderer. |
| The mapping list is empty | The renderer has no blendshapes, or none was assigned. Check you picked the face rather than the body, and that **Import BlendShapes** is enabled in the model's import settings. |
| Most rows say *None* | The mesh uses a different convention. Viseme or phoneme shapes (`A`, `E`, `viseme_aa`) cannot be driven at all; ARKit names under unusual decoration can be mapped by hand. |
| A blendshape popup shows `<missing: name>` | The stored name is not on the current mesh — usually a preset loaded onto a different rig. Pick the right shape. |
| The mouth moves but the expression is flat | Only jaw and mouth shapes were matched. Check the `brow`, `cheek`, `eye` and `nose` rows. |
| Barely any movement from this audio | The model animates speech. Music, effects, heavy noise and silence produce little movement. Try a clean speech recording to confirm the plugin itself works. |
| *"Animation data has no frames"* | The clip yielded no samples. Check it contains audio, and that its import **Load Type** lets `AudioClip.GetData` read it. |
| *"Bone 'X' is driven by two mappers"* | Keep every bone in a single mapper. |
| *"Eye bone 'X' is driven by both … and the eyes mapper"* | Remove that bone from the Face Blends Mapper; the Eyes Rotation Mapper owns it. |
| *"… is not a descendant of 'Root'. Its curves are skipped"* | The renderer or bone sits outside the Face Animator's hierarchy. Move the Face Animator to a common ancestor. |
| *"has no captured rest pose"* | A bone was assigned while posed. Return the rig to rest and press **Re-capture rest**. |
| The face is stuck in an expression in the Editor | A row's **Preview** slider or the eyes preview is still active. Collapse the row, press **Stop preview**, or select another object. |
| *"Invalid Path — Please save inside the Assets folder"* | The save panel pointed outside the project. |

</div>

### Emotions, eyes and blinking

<div class="table-scroll" markdown="1">

| Symptom | Cause and fix |
|---|---|
| The performance is unchanged after assigning an emotion preset | Emotions are a generation-time input. Generate again. |
| *"Every segment is skipped, so generation falls back to neutral"* | Every segment has an unknown emotion or a share of 0. Pick emotions from the popup and give each a share above 0. |
| *"The model has no emotion named X"* | Only the ten names listed above exist; matching is case-insensitive but spelling is not forgiven. |
| *"produced segments shorter than one audio sample"* | Too many segments for the clip length. Use fewer, or larger shares for the ones that matter. |
| Emotion transitions pop | Raise **Crossfade**; 0.25 s is a good default. |
| The eyes never move and the console says nothing | The Face Animator's **Eyes** field is empty. This case is silent by design — check the field. |
| *"has no eye bones assigned, so the eyes stay still"* | The component is assigned but the two **Bone** fields are not. |
| The eyes roll instead of turning, or one eye mirrors the other | Wrong **Pitch Axis** / **Yaw Axis** for this rig, or a mirrored bone. Calibrate with the Preview sliders, and use a **Scale** of −1 to reverse a direction. |
| The eyes look cross-eyed, or the eyeball clips through the lid | Lower **Look At Convergence**, raise **Gaze Symmetry**, or tighten the **Pitch / Yaw Limits**. |
| No blinking at all | `eyeBlinkLeft` / `eyeBlinkRight` are not mapped to real blendshapes, or **Enabled** is off. Neither case warns. |
| Every bake blinks at different moments | Set a non-zero **Random Seed**. |
| Eye contact is lost in a baked clip | A Look At target is sampled once at bake time. Do not bake moving eye contact. |
| *"the native plugin has no emotion support"* / *"no eye rotation support"* / *"returned no eye rotations"* | The C# scripts and `audio2face-unity.dll` are from different versions. Re-import the package, then run **Sync StreamingAssets**. Generation still works; only the named feature is missing. |

</div>

### Diagnostic log

When the console is not conclusive, the plugin can write its own and the native
SDK's output to a file — the only way to diagnose a failure on someone else's
machine, since the Unity console never sees what the SDK prints and a player
build has nowhere to put it. Off by default.

![The Diagnostic Log foldout in the TensorRT Engine window: a note on what the log contains, an unticked "Write a log file" checkbox, a Level dropdown set to Debug, the current log file path, and Change and Show buttons](/audio-face-animator/docs/img/tensorrt-diagnostic-log.png)

**In the Editor**, open **Tools → AudioFaceAnimator → Setup TensorRT Engine** and
expand **Diagnostic Log**. Tick *Write a log file* and pick a *Level* — `Debug`
logs every step, `Info` the main ones, `Error` only failures. *Change…* moves the
file, *Show* opens the folder, and the setting is remembered between sessions and
re-applied after every recompile, in append mode — so turn it off again when you
are done.

**From code**, called on the main thread before generation starts:

```csharp
using DevEloop.AudioFaceAnimator;

NativeLog.Enable(NativeLog.DefaultFilePath, NativeLogLevel.Debug);
// ... generate ...
NativeLog.Disable();
```

The default file is
`%USERPROFILE%\AppData\LocalLow\<company>\<product>\DevEloop\AudioFaceAnimator\audio-face-animator.log`.
Every line is flushed as it is written, so the log survives a crash. In a player
build, `Player.log` beside it carries everything the plugin logged, including the
engine-build progress.

## Support

Include your **GPU model**, **driver version**, **Unity version** and the **full
console output** — the messages identify the failure precisely, which makes most
issues diagnosable at a glance. If the console is not conclusive, turn on the
diagnostic log, reproduce the failure and attach the file.

<dev.eloop@outlook.com>
