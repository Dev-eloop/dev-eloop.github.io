---
layout: doc
title: Audio Face Animator documentation
description: Complete documentation for Audio Face Animator — the Unity plugin that generates ARKit facial animation from audio on your own machine, using the NVIDIA Audio2Face-3D SDK. Setup, character mapping, the scripting API and troubleshooting.
parent: Audio Face Animator
parent_url: /audio-face-animator/
---

**Audio Face Animator** generates facial animation from audio using the **NVIDIA
Audio2Face-3D** model, driving any character with **ARKit-compatible
blendshapes**. Everything runs on your own machine — no cloud service, no
account, no API key and no per-request cost, with the model, CUDA and TensorRT
all inside the package.

<ul class="doc-links">
  <li><a href="#requirements">Requirements</a></li>
  <li><a href="#editions">Versions comparison: Lite vs Full</a></li>
  <li><a href="#configuration">Configuration</a></li>
  <li><a href="#preparing-your-character">Preparing your character</a></li>
  <li><a href="#generate-animation-in-editor">Generate animation in Editor</a></li>
  <li><a href="#generate-animation-at-runtime">Generate animation at runtime</a></li>
  <li><a href="#troubleshooting">Troubleshooting</a></li>
  <li><a href="#support">Support</a></li>
</ul>

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

## Versions Comparison: Lite vs Full
{: #editions}

![Feature comparison: both versions generate facial animation from audio, process fully offline and work in the Unity Editor. Only the full version supports standalone builds and unlimited audio length; Lite is capped at 5 seconds of audio and animation](/audio-face-animator/docs/img/face-animator-versions-comparison.webp)

The Lite version has two limitations:

- **Audio length is limited at five seconds.**
- **It works in the Unity Editor only** — no standalone builds.

## Configuration

Both steps are prompted automatically when the package is imported, and both are
one-time. Use the menu items below to repeat them later.

### 1. Sync StreamingAssets

**Tools → AudioFaceAnimator → Sync StreamingAssets**, then **Copy All Files**. Run
it again after every plugin update.

![The StreamingAssets Sync window, listing model files missing from Assets/StreamingAssets/DevEloop/AudioFaceAnimator](/audio-face-animator/docs/img/streamingassets-sync.png)

This copies the model files to `Assets/StreamingAssets/DevEloop/AudioFaceAnimator`,
which is where Unity looks for them at runtime.

### 2. Setup TensorRT engine

**Tools → AudioFaceAnimator → Setup TensorRT Engine**, then **Build Engine for
This GPU**. About a minute, once per machine.

![The TensorRT Engine window: the detected GPU, a warning that no engine exists yet, and the Build Engine for This GPU button](/audio-face-animator/docs/img/tensorrt-engine-setup.png)

The engine is compiled from the `network.onnx` synced in step 1, because a
compiled engine only loads on the GPU architecture it was built for. Leave *Build
the full batch range declared by the model* off — it doubles the engine size for a
capability the plugin never uses.

In a standalone build the first generation call does this automatically, on each
player's machine.

## Preparing your character

Two components: a **Face Blends Mapper** on each face renderer, and one
**Face Animator** on the character that drives them.

### 1. Face Blends Mapper

Add it and assign your face `SkinnedMeshRenderer` to **Target Renderer**.

![The Face Blends Mapper component with no renderer assigned, showing the "Assign a SkinnedMeshRenderer with blendshapes" notice](/audio-face-animator/docs/img/face-blends-mapper-empty.png)

It immediately matches the mesh's blendshapes against the 52 ARKit shapes the
model drives — ignoring case, spaces and underscores, and accepting prefixed names
like `CC_Base_Body.jawOpen`.

**Add one mapper per face renderer** (head, teeth, tongue and eyes are often
separate).

![The Face Blends Mapper after auto-mapping, showing ARKit shapes matched to mesh blendshapes with weight sliders](/audio-face-animator/docs/img/face-blends-mapper-mapped.png)

Matching is deliberately generous, so read down the list once. Each row's
**dropdown** picks the mesh blendshape, or *None* to leave that shape undriven;
the **slider** is its maximum weight, 0–100. Weights are the tuning control — lower
`jawOpen` if the mouth opens too far. **Auto-Map Blendshapes** re-runs the matching,
discarding manual edits.

### 2. Face Animator

Add it to the character root and drag every mapper into its **Blends Mappers**
list. Any renderer whose mapper is not listed will not move.

![The Face Animator component with its Blends Mappers list expanded, showing five Face Blends Mapper references](/audio-face-animator/docs/img/face-animator-blends-mappers.png)

## Generate animation in Editor

![The Face Animator component in the Inspector, with Model, Blends Mappers, Audio Clip and the Generate, Play and Stop buttons](/audio-face-animator/docs/img/face-animator-inspector.png)

**Model** — which Audio2Face-3D model generates the animation: **James**,
**Mark** or **Claire**. Each was trained on a different performer and produces a
different speaking style. This is a *performance style*, not a character: it has
nothing to do with what your character looks like, and Claire will drive a male
character perfectly well. Try all three on the same audio.

**Audio Clip** — the speech to animate. No preparation needed.

Then **enter Play mode** — the preview plays audio through an `AudioSource`
created on Awake — press **Generate**, and press **Play** to preview the result.
**Stop** ends it early and resets the face to neutral. Generation runs on a
background thread, so the Editor stays usable.

On the very first generation the plugin may pause to compile a TensorRT engine,
logging progress to the console. Changing the AudioClip discards the generated
animation; changing the Model does not, so regenerate to hear it.

### Saving an AnimationClip

Once animation exists, a **Create AnimationClip** button appears. It writes a
standard Unity `AnimationClip` — one blendshape curve per mapped shape, per
renderer — which must be saved **inside your project's `Assets` folder**.

The clip has no dependency on the plugin, the SDK or an NVIDIA GPU, so it plays
from an Animator or Timeline on any platform Unity builds for. Two things to
know:

- **Curve paths are relative to the GameObject holding the Face Animator**, so
  moving that component to a different level of the hierarchy invalidates the
  clip. Put it on the character root and keep it there.
- **Weights are baked in** when the curves are written. Changing a mapper weight
  afterwards means generating and saving again.

### The sample scene

`Assets/DevEloop/AudioFaceAnimator/Scenes/AudioFaceAnimator.unity` has all three
characters set up, a model selector, generation controls and a file browser for
loading a WAV from disk. It is the quickest way to confirm the plugin works on
your machine, and a working reference for wiring the API to UI.

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
`IsGenerating` to disable your UI while it runs, as the sample scene does.

<div class="table-scroll" markdown="1">

| Member | What it does |
|---|---|
| `AudioClip AudioClip { get; set; }` | The clip to animate. Assigning a different clip discards the generated animation. |
| `AudioFaceModel model` | `James`, `Mark` or `Claire`. |
| `List<FaceBlendsMapper> blendsMappers` | The mappers to drive. |
| `Task GenerateAsync()` | Generates on a background thread. Await it. |
| `void Play()` / `void Stop()` | Play the animation with its audio; stop and reset to neutral. |
| `AnimationClip CreateAnimationClip(string name = "")` | Builds a clip from the current animation, or `null` if nothing was generated. |
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

### Shipping a standalone build

**Keep `network.onnx` in StreamingAssets and let the first generation call build
an engine on each player's machine.** Your players' GPUs are not yours, and the
engine built on your machine will not load on most of theirs. There is no way to
build one engine covering many architectures, so this is the only approach that
works everywhere. The build takes about a minute, happens once per machine, and
is cached to `Application.persistentDataPath`, which stays writable even when the
game is installed somewhere read-only.

A minute of silence looks like a hang, so trigger that first generation somewhere
you control — a loading screen, or before a character first speaks — and show
your own progress UI. Build progress is also written to the player log.

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

Open the Unity console first. Since version 1.1 every failure reports the actual
reason, including what TensorRT itself said.

<div class="table-scroll" markdown="1">

| Symptom | Cause and fix |
|---|---|
| *"No usable CUDA device"*, or the native plugin could not be loaded | No NVIDIA GPU was found, or the driver is too old. Update to driver 576 or newer. On switchable-graphics laptops, make sure Unity runs on the NVIDIA GPU. |
| *"The TensorRT engine … cannot be loaded on this GPU"* | A `network.trt` you supplied was built for a different architecture. Build one via **Setup TensorRT Engine**, or remove the file and let the plugin build from `network.onnx`. |
| *"… is not a TensorRT engine (N bytes)"* | A `network.trt` did not arrive intact — usually Git LFS storing a pointer instead of the file. Delete it and rebuild. |
| *"no execution context could be created"* | Not enough free VRAM. Close other GPU-heavy applications. If you enabled the full batch range, rebuild with it off. |
| *"network.onnx is missing"* | Run **Tools → AudioFaceAnimator → Sync StreamingAssets**. |
| The engine build fails or is very slow | Building needs several GB of free VRAM and disk. Close other GPU applications; the console shows what TensorRT reported. |
| Animation is capped at 5 seconds | You are running the Lite edition — the full version is [on the Asset Store](https://assetstore.unity.com/packages/slug/383624). If you bought the full version, the model files did not sync: run **Sync StreamingAssets**. |
| Generation succeeds but nothing moves | The character is not mapped. Check the mappers are listed in **Blends Mappers**, and that their rows are not all *None*. |
| The mapping list is empty | The renderer has no blendshapes, or none was assigned. Check you picked the face rather than the body, and that **Import BlendShapes** is enabled in the model's import settings. |
| Most rows say *None* | The mesh uses a different convention. Viseme or phoneme shapes (`A`, `E`, `viseme_aa`) cannot be driven at all; ARKit names under unusual decoration can be mapped by hand. |
| The mouth moves but the expression is flat | Only jaw and mouth shapes were matched. Check the `brow`, `cheek`, `eye` and `nose` rows. |
| One shape is driven by the wrong curve | Two ARKit rows point at the same mesh blendshape, so the lower one wins. Scan for duplicates and set one to *None*. |
| Barely any movement from this audio | The model animates speech. Music, effects, heavy noise and silence produce little movement. Try a clean speech recording to confirm the plugin itself works. |

</div>

### Diagnostic log

When the console is not conclusive, the plugin can write its own and the native
SDK's output to a file — the only way to diagnose a failure on someone else's
machine. Off by default.

**In the Editor**, open **Tools → AudioFaceAnimator → Setup TensorRT Engine** and
expand **Diagnostic Log**. Tick *Write a log file* and pick a *Level* — `Debug`
logs every step, `Info` the main ones, `Error` only failures. *Change…* moves the
file, *Show* opens the folder, and the setting is remembered between sessions.

**From code**, called on the main thread before generation starts:

```csharp
using DevEloop.AudioFaceAnimator;

NativeLog.Enable(NativeLog.DefaultFilePath, NativeLogLevel.Debug);
// ... generate ...
NativeLog.Disable();
```

Every line is flushed as it is written, so the log survives a crash.

## Support

Include your **GPU model**, **driver version**, **Unity version** and the **full
console output** — the messages identify the failure precisely, which makes most
issues diagnosable at a glance. If the console is not conclusive, turn on the
diagnostic log, reproduce the failure and attach the file.

<dev.eloop@outlook.com>
