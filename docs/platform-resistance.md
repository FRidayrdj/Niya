# Platform resistance

**What a mobile OS actually does to an always-on assistant, and what it costs the user.**

Android was not built for a process that wants the microphone for hours while nobody is looking at it. A dozen manufacturers have each hardened their stack against exactly that, mostly for battery reasons and partly for the assistant they ship themselves. Most of what follows is written down nowhere. Some of it contradicts the documentation.

All of it was observed on real hardware: Samsung One UI, OnePlus OxygenOS, Xiaomi HyperOS and MIUI. Each section takes the same four questions in turn. What the platform does, how the failure presents from inside the app, what the app does about it, and who pays.

One thing to be clear about before any of it. None of this defeats a permission. The user granted the microphone. What follows is where the operating system and the app disagree over what that grant covers.

---

## 1. The microphone is granted to a window, not to a process

### What the platform does

`RECORD_AUDIO` gets checked at capture time against something closer to "does this process have visible foreground UI" than to "did the user grant this app microphone access". A foreground service does not satisfy it. The service is running, the notification is showing, the grant is on record, and the capture path comes back empty.

Of everything on this page, this behaviour matters most, because the entire premise of a wake-word assistant is that it works when nothing is on screen.

### How it shows up

No exception gets thrown. `AudioRecord` constructs, `startRecording()` returns cleanly, `read()` hands back exactly as many samples as you asked for, and all of them are zero. The framework reports success throughout.

The first detector treated eighty consecutive chunks below the noise floor as a dead microphone and tried to fix it by reopening the handle. That loop ran forever. The handle was never the problem.

The only trustworthy signal sits downstream: if the mel-spectrogram stage has never emitted a frame, the microphone is not open, whatever the API says. The detector surfaces that as a callback, and the reply path uses it to know when audio is genuinely live rather than merely claimed.

### What the app does

Two approaches, chosen by what the device will tolerate.

**Send the audio somewhere the rule doesn't reach.** Bluetooth hands-free capture (SCO/HFP) is classified as telephony input, and whatever arbitration governs the built-in microphone does not govern a phone call. With the screen off, the app holds SCO open permanently and runs detection, recognition and replies entirely over the headset. Not a privilege escalation. A phone call that never ends.

**Make a window exist.** Where the built-in microphone is needed (screen on, where its quality is better), the app briefly presents a zero-pixel window. A `0x0` window renders nothing and intercepts nothing, but as far as capture arbitration is concerned it is a visible foreground window, and the microphone comes back.

The naive version of that second trick is worse than skipping it, which is worth spelling out. On OxygenOS, a transparent Activity in its own task has nothing behind it to show through, so the compositor paints black strips that swallow touches. On OnePlus, creating a new task drops the main Activity out of the resumed state and touches in the centre of the screen go missing for several seconds while the transition settles. The status bar and the navigation buttons keep responding the whole time, so it reads as an app bug rather than an input stall.

Three countermeasures, then:

1. The window is `0x0`, not merely transparent. Nothing renders, so nothing can intercept.
2. The not-touchable and not-focusable flags go on in `onWindowFocusChanged`, after the window has attached. Set before `super.onCreate()` they leave the compositor mid-handshake: it offers focus to a window that has already declared it will never take focus, the handoff never finishes, and only a full input recomputation (pressing Home) clears it.
3. The periodic self-finish is mandatory rather than defensive. It was removed once during development, and without it the unlock task sat above the main Activity for an entire Bluetooth session, leaving input dead for the session instead of for the moment of a rebuild.

OnePlus takes a third route: a zero-sized `WindowManager` overlay owned by the service. No Activity, no task, no activity-manager transition, so nothing can take focus away in the first place. It needs the "display over other apps" toggle, which is a settings screen rather than a runtime dialog, so the Activity path stays as the fallback when the toggle is off.

### What it costs the user

The overlay permission, on OnePlus. A brief capture gap at every route change. And an implementation that has to know what phone it woke up on, because `Build.MANUFACTURER.equals("oneplus")` is a real branch in shipping code.

---

## 2. After playback ends, the capture path can stay reserved

### What the platform does

Some audio stacks don't release the capture path when A2DP playback finishes. It stays held, apparently reserved for the manufacturer's own assistant, which is waiting for the same cue: is the user about to talk to me. Recreating the `AudioRecord` doesn't reclaim it. Toggling audio focus doesn't either. Waiting doesn't help.

Nothing documents this, and it doesn't reproduce everywhere, which is what makes it expensive. It looks like a race, or like your bug, and it only appears in a Bluetooth workflow after the assistant has spoken.

### How it shows up

The dead-microphone signature again, a handle that opens and never delivers samples, but only after a reply has played, and only on A2DP. The in-place restart doesn't help. The escalation to a full rebuild doesn't help either, because a fresh handle lands on the same reserved path.

### What the app does

It stops trying to fix the microphone and fixes the state instead. The routing machine keys on screen state, because screen state is what the platform's arbitration keys on:

| Screen state | Capture | Recognition | Replies |
|---|---|---|---|
| **Off**, headset connected | SCO/HFP held permanently | Over SCO | Over the hands-free stream |
| **On**, headset connected | Built-in mic via `VOICE_RECOGNITION` | SCO on demand, for the recognition step only | A2DP |
| **Either**, no headset | Built-in mic only while a window exists | Same | Speaker |

Transitions are explicit, never inferred. Screen off with a headset tears down the built-in detector and brings SCO up for always-on listening. Screen back on reverses it. The screen-on interaction is a four-step handshake: the wake word fires, SCO is requested, the SCO broadcast triggers recognition, and only once recognition *and* the reply have both finished does SCO come down and the built-in detector rebuild.

An SRF 14-byte `AudioRecord` with an `AudioSource` of `MIC` behaves differently enough on the SCO path that the detector takes the choice as a constructor argument rather than branching internally. Sample rate negotiation differs too. One device delivers 8 kHz over SCO, so the pipeline upsamples to 16 kHz by linear interpolation before the mel stage. Skip that and the embedding stage produces noise without complaining.

### What it costs the user

Reliable screen-off listening effectively requires a Bluetooth headset. Without one there is no capture path the platform cannot deny, and the app says so instead of pretending.

There is also a perceptible order to events: wake word, SCO negotiation, the recogniser's own listening cue. The app used to play a cue of its own before all that, and removed it once it became clear the sound landed about two and a half seconds before the microphone was actually open, teaching users to speak too early. The platform's cue fires at capture, which is the only honest signal, and the wake-to-listen latency dropped by that delay as a side effect.

---

## 3. Another app taking the mic can fire a phantom wake word

### What the platform does

When a different client opens the microphone (the camera recording video, a voice recorder, anything at all), the audio HAL reconfigures the capture path. The reconfiguration pushes a burst of discontinuous frames into whatever else is reading. To a rolling-window classifier, a discontinuity is not obviously distinguishable from a sound.

This one is a compositing bug rather than a policy. No permission was denied. The glitch burst simply scored above threshold.

### How it shows up

Samsung, specifically. Starting a video recording fired a false wake word, the assistant launched speech-to-text, that took the microphone, and the user's recording died. From the outside it looks like the assistant sabotaging the camera, and it is the sort of interaction that ends with an uninstall.

### What the app does

It watches for the situation instead of trying to detect the glitch. `AudioManager.AudioRecordingCallback` reports every change to recording configuration, so the app can see another client's session appear, and on API 29+ it can see that its own stream has been client-silenced, meaning the framework is feeding it zeros because a foreground app owns the mic.

While another client holds the capture path, scoring is suppressed and the pipeline buffers are flushed. When that client lets go, the app flushes again and holds a short cooldown, since the release-side routing churn produces the same burst as the acquire side. Both edges need the guard.

One detail needed a second pass. While suppression is active, the dead-microphone counter has to be frozen. A deliberately silenced stream reads as pure zeros, which is precisely the dead-microphone signature, so without the freeze any video recording longer than about 6.4 seconds tripped the eighty-silent-chunk rebuild loop for its whole duration. The app spent the recording rebuilding detectors that were never broken.

The in-call guard is separate and older, because a phone call holds the device in `MODE_IN_COMMUNICATION`. Camera recording never does, which is why the call guard is blind to it.

### What it costs the user

The assistant is deaf while another app records. That is the correct behaviour and still should be silent rather than surprising. It also means the detector needs a foreground-app signal it was never designed to have.

---

## 4. MIUI's Autostart is not a permission

### What the platform does

On Xiaomi-family devices (Xiaomi, Redmi, POCO, Black Shark) MIUI's Autostart restriction can silently stop an app from listening in the background or receiving notification access. No public Android permission covers it and there is no API. `AutoStartManagerService` rejects binding the notification listener even when standard Notification Access is granted. App code cannot override it.

### How it shows up

Nothing throws. The service isn't started, or the listener isn't bound, or it was bound and then isn't. The app is installed, permissioned, running, and hears nothing.

### What the app does

The only thing available: tells the user. A one-time setup screen, shown only on MIUI-family devices (keyed on `Build.MANUFACTURER` and `Build.BRAND` across all four brands, not on the "Xiaomi" string alone), deep-links into the Autostart settings and explains the toggle. The battery-optimisation step on the same screen uses the real `PowerManager` API and can report its actual state. Autostart cannot, so the UI says "Opened — make sure it's ON" rather than inventing a granted state it has no way to verify.

The intent component name is undocumented and has moved between MIUI and HyperOS versions and regions. The deep link has a `try`/`catch` fallback to the app's own details screen for ROMs where it doesn't resolve.

### What it costs the user

A setup step most people have no reason to know exists, surfaced at the only moment it is useful, and an app that quietly fails if they skip it.

---

## 5. Rolling-window classifiers need the quiet parts

Not a platform behaviour at all, though it presented as one for a while, which is why it belongs on this page.

### What happened

The detector had a cheap optimisation: skip any chunk whose peak amplitude fell below a small threshold, on the reasoning that silence cannot contain a wake word. On a healthy microphone the ambient floor keeps peaks just above that threshold, so the gate never fired. Embeddings ran continuously through silence and everything worked.

On an attenuated capture path, a phone-mic fallback after a failed SCO connect on OnePlus, silence reads as near zero. Every quiet chunk got dropped, and the consequences chained:

- embeddings fired only while the user was speaking;
- the sixteen-embedding score window never filled contiguously, so nothing scored at all;
- the wake word went deaf;
- and eighty consecutive quiet chunks (~6.4 s) tripped the dead-mic detector and then the rebuild loop, producing a bursty stall that looked like a hardware fault.

### The fix

Feed every chunk to the mel/embed pipeline, silence included. openWakeWord is a classifier over a continuous stream, not over utterances, and a gap is not "nothing to classify" but a corrupted window. Amplitude is now used only for the dead-microphone counter and the microphone-is-alive signal, never to decide what reaches the model.

That is also why the guard has to freeze during external capture. Silence carries four different meanings at once: a quiet room, an attenuated path, another app owning the mic, and a dead handle. The app has to know which one it is looking at.

---

## What the whole thing costs

Before deciding this way of building the assistant is worth it, the bill:

- **A real architectural split by screen state.** Two capture paths, two audio sources, two sample rates, two handoffs, and a state machine that has to be right or the microphone ends up in the wrong hands.
- **A window workaround on a platform where "background" is supposed to mean something.** A zero-pixel Activity and a zero-sized overlay are both faintly absurd. Both exist because the grant is arbitrated against UI the user cannot see.
- **Manufacturer branches in shipping code.** OnePlus gets an overlay, everyone else gets an Activity. Xiaomi gets a setup screen, everyone else gets nothing. Three paths where the ideal is one.
- **A deaf assistant while other apps record.** Correct, and still a surprise.
- **Setup instructions nobody should have to read.** Autostart, battery optimisation, display over other apps. Each one is a place to lose a user who only wanted to talk to their phone.

The alternative is to call an always-on assistant on Android a best-effort feature, working on some phones and quietly not on others. The workarounds here are what it cost to refuse that.

---

*Tested on Samsung One UI, OnePlus OxygenOS and Xiaomi HyperOS across Android 10–16. Behaviour described here is observed, not documented, and the platform promises nothing about the next release.*
