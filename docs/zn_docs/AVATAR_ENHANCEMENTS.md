# Avatar Enhancement Roadmap

Progress tracker for the 3D avatar system improvements.
**Ultimate goal:** generate an avatar from a real teacher image that looks, moves, and sounds like them.

---

## Status Key

- `[ ]` Not started
- `[~]` In progress
- `[x]` Done

---

## Tier 1 — Easy (days, no new tech)

### [x] 1. Better TTS voices

Replace `window.speechSynthesis` with a TTS API (OpenAI TTS, ElevenLabs, Azure) for natural-sounding voices.
**Impact:** Single highest-impact change for perceived realism.
**Files:** `ui/src/lib/components/chat/AvatarChat.svelte` → `speakText()`

---

### [x] 2. LLM-annotated animation instructions ← **DONE**

Instead of post-hoc keyword sentiment analysis, the LLM itself returns a structured JSON block
with explicit animation codes per response. The avatar's expression, gestures, head movement,
and GLB animation are now **intentional** rather than guessed.

**Changes made:**

- `ui/src/routes/avatar-talk/+page.svelte`: New comprehensive system prompt with full animation
  reference table. Raw JSON response is now passed directly to `AvatarChat` (no stripping).
- `ui/src/lib/components/chat/AvatarChat.svelte`: Added `boardText` internal variable so the
  classroom board shows clean readable text, not raw JSON.

**JSON format the LLM now returns:**

```json
{
  "response": "Text for the avatar to speak",
  "animation": {
    "facial_expression": 1,
    "head_movement": 1,
    "hand_gesture": 4,
    "eye_movement": 0,
    "body_posture": 0
  },
  "glbAnimation": "talking_happy",
  "glbAnimationCategory": "expression"
}
```

**Animation code reference:**
| Field | Codes |
|---|---|
| `facial_expression` | 0=neutral 1=smile 2=frown 3=raised_eyebrows 4=surprise 5=wink 6=sad 7=angry |
| `head_movement` | 0=no_move 1=nod_small 2=shake 3=tilt 4=look_down 5=look_up 6=turn_left 7=turn_right |
| `hand_gesture` | 0=no_move 1=open_hand 2=pointing 3=wave 4=open_palm 5=thumbs_up 6=fist 7=peace_sign 8=finger_snap |
| `eye_movement` | 0=no_move 1=look_up 2=look_down 3=look_left 4=look_right 5=blink 6=wide_open 7=squint |
| `body_posture` | 0=neutral 1=forward_lean 2=lean_back 3=shoulders_up 4=rest_arms 5=hands_on_hips 6=sit 7=stand |
| `glbAnimation` (expression) | talking_neutral talking_happy talking_excited talking_thoughtful talking_concerned expression_smile expression_sad expression_surprise expression_thinking expression_angry |
| `glbAnimation` (idle) | idle_normal idle_shift_weight idle_look_around idle_default idle_stretch idle_impatient |

---

### [ ] 3. Idle eye tracking / gaze toward camera

Add subtle eye-socket bone rotations that drift back toward the camera between blinks.
**Impact:** Makes the avatar feel like it's making eye contact.
**Files:** `AvatarChat.svelte` → `startEnhancedIdleAnimation()`

---

### [ ] 4. Configurable animation personality sliders

Expose `ANIMATION_SETTINGS` constants as UI controls per avatar — gesture intensity,
expression duration, breathing depth.
**Files:** New settings panel component + `AvatarChat.svelte` props

---

### [ ] 5. Full ARKit 52-blendshape export

Re-export the 4 GLB avatar models from Ready Player Me with the full ARKit blendshape set.
Unlocks cheek puffs, brow raises, eye squints etc. — zero code change needed, the existing
`performExpression()` picks them up automatically.
**Files:** Asset pipeline only (Ready Player Me editor → re-export GLBs)

---

## Tier 2 — Medium (weeks, new API integrations)

### [ ] 6. Accurate lip sync via TTS viseme events

Azure Cognitive Speech SDK and ElevenLabs both return phoneme/viseme timing alongside audio.
Feed those timestamps into `animateMouth()` for perfect lip sync instead of syllable estimation.
**Requires:** Backend proxy endpoint for audio + viseme data stream.

---

### [ ] 7. Voice cloning per teacher

Teacher records 30–60s of speech → ElevenLabs Voice Cloning or Azure Neural TTS Custom Voice
generates their voice ID. Stored on teacher profile. All avatar TTS sounds like the real teacher.
**Requires:** Backend `voice_id` field on teacher profile + ElevenLabs/Azure integration.

---

### [ ] 8. Ready Player Me photo-to-avatar API ← **KEY MILESTONE**

Teacher uploads one photo → RPM API generates a personalized GLB avatar matching their
appearance (skin tone, face shape, hairstyle). Plugs directly into the existing Three.js loader.
**Requires:** Backend endpoint to call RPM API + store returned avatar URL per teacher.
**This is the most practical first step toward "avatar from teacher image."**

---

### [ ] 9. WebRTC session recording

Record avatar sessions (TTS audio + animation metadata) for async playback / tutoring content.

---

### [ ] 10. Improved Three.js shading

Custom ShaderMaterial with rim lighting + skin subsurface scattering + post-processing bloom
via Three.js `EffectComposer` + `UnrealBloomPass`.
**Files:** `AvatarChat.svelte` → `initThreeJs()` renderer setup

---

## Tier 3 — Hard (months, ML infrastructure)

### [ ] 11. Real-time face tracking → expression mirroring

MediaPipe Face Landmarker (WASM, runs in-browser) extracts 52 ARKit blendshapes from the
teacher's webcam at 30fps. Those coefficients drive the avatar's morph targets in real-time.
The avatar mirrors the teacher's live expressions.
**Requires:** `@mediapipe/tasks-vision` package + webcam permission flow + 60fps render budget.

---

### [ ] 12. 2D photo-realistic talking head (HeyGen / D-ID)

Replace Three.js canvas with a photorealistic `<video>` element generated by HeyGen or D-ID
from a teacher photo + TTS audio. Best for async content; not suitable for real-time interaction.

---

### [ ] 13. NeRF / Gaussian Splatting from photo set

20–50 teacher photos → Instant NGP or 3DGS reconstructs a photorealistic 3D representation.
Currently static (not animatable). Research tools like GaussianAvatars are emerging.
**Timeline:** 12–18 months from being production-deployable.

---

### [ ] 14. Full neural avatar pipeline

Teacher uploads one photo → rigged animatable 3D avatar matching their appearance, driven
by voice-cloned TTS + accurate lip sync + real-time expression mirroring from webcam.
Combines steps 6 + 7 + 8 + 11. Commercial APIs (Synthesia, HeyGen streaming avatar) are
converging on this. This is the end goal.

---

## Recommended Phasing

```
Phase 1 (now):    #2 LLM animation prompts ✓ + #1 better TTS
Phase 2 (next):   #8 RPM photo-to-avatar + #7 voice cloning
                  → teachers upload photo + voice → avatar looks and sounds like them
Phase 3 (later):  #6 accurate lip sync + #11 webcam expression mirroring
                  → avatar is a real-time mirror of the teacher
Phase 4 (future): #14 neural avatar pipeline when API quality/latency allows
```
