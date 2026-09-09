# taudplay

**Play `.taud` songs — in a browser tab, or in Node. No tracker attached.**

`taudplay` is the playback half of the [Microtone](https://microtone.cc)
tracker, cut down to the part a *listener* needs. It is the same engine, note
for note and sample for sample: a file rendered here is bit-identical to the
same file rendered in the tracker.

What it exposes is deliberately small — the whole API is a transport, **one
fader per voice**, and **two probes per voice**:

| | |
|---|---|
| **Knob** | `setVoiceGain(voice, gain, fadeMs)` — 1 = as written, 0 = silent |
| **Probe** | `getVoiceVolume(voice)` — how loud that channel is right now, 0…1 |
| **Probe** | `getVoicePan(voice)` — where it sits, 0 (left) … 0.5 … 1 (right) |

That is the point. A game does not want a pattern editor; it wants to duck the
lead when the player enters a cave, bring the drums up in combat, and draw a
little dancing meter on the pause screen. A tracker song is 32 or 64
independent channels of music that were *written together* — fading them
against each other gives you contextual scoring for the cost of one file.

## Install

```
npm install taudplay
```

…or just copy `src/` in. There is no build step and no dependency to install:
everything is ES modules, and the two vendored decompressors are single files.

## Play something

```js
import { TaudPlayer } from "taudplay";

const player = await new TaudPlayer().init();
const songs = await player.load(await (await fetch("theme.taud")).arrayBuffer());

// Browsers only start audio from a user gesture.
button.onclick = async () => { await player.resume(); player.play(); };
```

### Mix it

```js
player.setVoiceGain(4, 0.0, 1200);   // fade channel 5 out over 1.2 s
player.setVoiceGain(4, 1.0, 400);    // …and back in, faster

// Duck everything but the drums.
for (let v = 0; v < player.channelCount; v++) {
  if (v !== DRUMS) player.setVoiceGain(v, 0.3, 800);
}
```

The fade is applied inside the audio worklet, once per rendered block (every
2.7 ms at 48 kHz), so a slow fade is smooth without your game loop driving it.
A voice's NNA ghosts and metainstrument layer children follow its fader, so a
faded channel really does take everything it spawned with it.

### Watch it

```js
function frame() {
  for (let v = 0; v < player.channelCount; v++) {
    bars[v].style.height = `${player.getVoiceVolume(v) * 100}%`;
    bars[v].style.left = `${player.getVoicePan(v) * 100}%`;
  }
  requestAnimationFrame(frame);
}
```

Both probes are read out of a snapshot the worklet posts about every 16 ms, so
reading them costs nothing and they never block the audio thread.

### Use your own graph

```js
await player.init({ context: myAudioContext, destination: myReverbSend });
```

## …and in Node

`TaudRenderer` is the same library without Web Audio: you pull blocks instead
of the sound card pulling them, and the probes describe the state at the end of
each block.

```js
import { readFile, writeFile } from "node:fs/promises";
import { TaudRenderer } from "taudplay";

const r = new TaudRenderer(await readFile("theme.taud"));
r.setVoiceGain(4, 0);                      // bounce the song without channel 5
await writeFile("theme.wav", r.toWav(120));
```

Or drive it block by block and automate the mix as it renders:

```js
r.play();
let pcm = r.render(30, (rr, frame) => {
  if (frame === 48000 * 8) rr.setVoiceGain(4, 0, 2.0);  // fade at 0:08
});
```

## API

### `TaudPlayer` (browser)

- `await init({ context, destination, snapshotIntervalMs, workletUrl })`
- `await resume()` / `await close()` — `running`, `sampleRate`
- `await load(bytes)` → song descriptors; `songs`, `title`, `info`
- `selectSong(i)`, `play()`, `stop()`, `seekCue(n)`, `setVolume(0…1)`
- `setBinaural(on)` — head-model monitoring for surround songs
- `setVoiceGain(v, gain, fadeMs)`, `getVoiceGain(v)`
- `getVoiceVolume(v)`, `getVoicePan(v)`
- `playing`, `cue`, `row`, `bpm`, `speed`, `channelCount`
- `onSnapshot`, `onLoaded` callbacks

### `TaudRenderer` (anywhere)

The same knob, probes and transport, plus `renderChunk()`, `render(seconds,
onChunk)` and `toWav(seconds, { sampleRate })`.

## What is *not* here

No pattern or instrument editing, no document model, no undo, no import
converters, no jam keyboard, no analysis or loudness metering, no stem or
ambisonic export, and no per-voice observability beyond those two numbers. All
of that lives in [Microtone](https://microtone.cc), which is where songs are
made. This library only plays them.

Songs are made with **Microtone** — a tracker for the notes a piano cannot
play, free and in your browser at [microtone.cc](https://microtone.cc).

## Format support

Full `.taud` files, any format version the engine reads, 32- or 64-channel,
stereo or surround. `.tsii` (samples and instruments) and `.tpif` (a single
pattern) carry no song and are rejected.

## Licence

Copyright (C) 2026 CuriousTorvald.

LGPL-3.0-or-later — see `COPYING`. You may link this library into a
proprietary application; changes *to the library itself* must be shared.

The vendored decompressors keep their own (MIT) licences:
[fflate](https://github.com/101arrowz/fflate) and
[fzstd](https://github.com/101arrowz/fzstd). The binaural filter set in
`src/engine/hrir-sadie.js` is the GoogleVR/SADIE set, Apache-2.0.

---

Generated from Microtone.js 2.44.0 (engine 9f8917cdf676) by
`tools/make-taudplay.js`. Do not edit the engine here — edit it there and
regenerate, or the library and the tracker stop agreeing about what a song
sounds like.
