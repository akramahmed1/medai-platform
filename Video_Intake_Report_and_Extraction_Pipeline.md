# TRAINING VIDEO INTAKE REPORT & CONTENT EXTRACTION PIPELINE
**Internal — for Akram / build agents only. Do not share.**
Date: July 2026 · Status: metadata verified; content analysis pending transcription (see §3)

---

## 1. Verified asset metadata

| | Video 1 | Video 2 |
|---|---|---|
| File | NAIP_Radiologist_Training___25th_May_2026.mp4 | NAIP_Radiologist_Training_GPHC___18th_June_2026.mp4 |
| Duration | 2h 49m 19s (10,159s) | 1h 57m 53s (7,073s) |
| Size | 390 MB | 300 MB |
| Video | H.264, 1110×720 | H.264 (same profile family) |
| Audio | AAC | AAC |
| Type (from format/naming) | Recorded live training session | Recorded live training session |
| Combined raw material | **4h 47m** — enough to seed ~2 full courses after segmentation | |

Honest analysis status: frames were sampled every 150s (116 frames, 8 contact sheets). This confirms the recordings are intact and screen-share-based, but **no content claims (topics, slide text, cases, speakers) are made in this report** — spoken audio carries the substance of a training session and could not be transcribed in this environment (no STT tooling, no network). Anything more specific would be guesswork, and guesswork must not become course content.

## 2. Immediate handling rules (before any processing)

1. **Confidential source material.** Store only in the private Supabase `videos/` bucket or local disk — never YouTube, never public URLs, never third-party transfer tools.
2. **Patient-data screening is mandatory before LMS use.** Radiologist training recordings very likely show real studies on screen. Before any clip enters a course: review for patient names/IDs/DOBs in the viewer UI, burned-in DICOM overlays, or spoken identifiers in audio. Flag → trim/blur/re-record those segments. This enforces the platform's no-patient-data red line at the content source.
3. Filenames themselves reference program/site context — rename working copies to neutral IDs (`course-src-001.mp4`, `course-src-002.mp4`) before they touch the repo or storage, consistent with the codebase neutrality rule.

## 3. Content extraction pipeline (run in Claude Code — has network + can install Whisper)

**Step 1 — transcribe (local, nothing leaves the machine):**
```bash
pip install faster-whisper
python - <<'PY'
from faster_whisper import WhisperModel
m = WhisperModel("medium", compute_type="int8")
for f in ["course-src-001.mp4", "course-src-002.mp4"]:
    segs, _ = m.transcribe(f, vad_filter=True)
    with open(f + ".transcript.txt", "w") as out:
        for s in segs:
            out.write(f"[{int(s.start)//3600:02d}:{int(s.start)%3600//60:02d}:{int(s.start)%60:02d}] {s.text.strip()}\n")
PY
```
(`medium` balances accuracy/speed on CPU; use `large-v3` if the machine has a GPU. Accented English handles well; review low-confidence sections manually.)

**Step 2 — visual index (slides/screens at 1-minute grain):**
```bash
mkdir -p frames && ffmpeg -i course-src-001.mp4 -vf "fps=1/60,scale=1110:-1" -q:v 3 frames/f_%04d.jpg
```

**Step 3 — segmentation & content report (Claude, human-approved):** feed each transcript to Claude with this prompt skeleton:
> "This is a timestamped transcript of a recorded radiology-software training session. Produce: (1) a session content report — topics in order with start/end timestamps, workflows demonstrated, Q&A themes; (2) proposed module boundaries for an LMS (5–15 min per lesson), each with title, learning objective, and exact cut timestamps; (3) 5 quiz questions per proposed module drawn ONLY from transcript content, with answers and timestamp citations; (4) a redaction flag list — any timestamps where names, patient identifiers, or sensitive details are spoken."

Cross-check module boundaries against the Step-2 frames (slide changes ≈ natural cut points). **Nothing publishes without human approval** — this is the §P3 authoring rule, live.

**Step 4 — cut & compress approved clips:**
```bash
ffmpeg -i course-src-001.mp4 -ss <start> -to <end> -c:v libx264 -crf 26 -vf scale=1280:-2 -c:a aac -b:a 96k lesson-01.mp4
```
Target ≤ ~40MB per 10-min lesson for the low-bandwidth requirement.

**Output of this pipeline = the detailed content report you asked for**, grounded in the actual audio — plus the flagship course's lessons, quizzes, and redaction list as a by-product. On a normal laptop, transcription of both files runs roughly 1–3 hours unattended; the Claude segmentation pass is minutes.

## 4. What this feeds

Transcripts + approved module map → constitution Task 4 (lesson player content) and Task 8 (assistant knowledge base). The two source recordings map naturally to two seeded courses (one per session) pending the content report.

---

## 5. LANGUAGE & VOICE STANDARD (added per content requirement)

**Requirement:** All video audio must be in clear, neutral Global English — internationally intelligible across Guyana, Singapore, Kenya, Philippines, UK, and other deployment regions. The original recordings are in Indian English and must be re-voiced before any video enters the platform.

**Standard to target:** BBC World Service / WHO training video English — clear, measured pace, neutral accent, no regional idioms.

### Demo path (fastest): AI dubbing

1. Complete Steps 1–3 (transcription + segmentation + human approval of transcript).
2. Clean the approved transcript: remove filler words, fix any transcription errors, ensure medical terminology is spelled correctly.
3. Upload cleaned transcript to **ElevenLabs** (elevenlabs.io) or **HeyGen** (heygen.com):
   - ElevenLabs: use the "Dubbing" feature with a neutral English voice (e.g. "Rachel" or "George" voices). Upload video + transcript. Download re-voiced MP4.
   - HeyGen: Video Translation feature — upload video, select target language "English (British)" or "English (US neutral)".
4. Review the output clip: check for awkward pauses, mispronounced medical terms (especially "Augmento", "PACS", "trachea", etc.), and overall pace.
5. For any clip with the speaker visibly on camera: check lip-sync. If unacceptable, use Option 2 below.
6. Upload approved re-voiced MP4 to Supabase `videos/` bucket via the server-side upload action.

**Estimated cost:** $50–200 per hour of video. Both videos combined (~4h47m) ≈ $150–500 total.
**Estimated time:** 2–3 days including review.

### Pilot path (best quality): Professional voice-over

1. Use the approved segmented transcripts (one per lesson).
2. Engage a professional medical voice-over artist (search "medical e-learning voice over" on voices.com or Voice123). Specify: neutral British or International English, medical training content, clear and measured pace.
3. Provide the script with pronunciation guide for any technical terms.
4. Voice artist records one audio file per lesson segment.
5. Editor syncs audio to video (any video editor; DaVinci Resolve is free).
6. Upload to Supabase.

**Estimated cost:** $200–500 per finished hour. Full library ≈ $1,000–2,500.
**Estimated time:** 1–2 weeks per video.

### What NOT to do

- Do not upload the original-accent audio to the platform — it is the first thing international reviewers notice.
- Do not use auto-translation (the content is already in English; this is accent/clarity, not translation).
- Do not skip the human transcript review step before re-voicing — AI dubbing inherits any transcription errors.

### Medical terminology pronunciation guide (for voice-over brief)

| Term | Pronunciation note |
|---|---|
| Augmento | aw-MEN-toe |
| PACS | "packs" (one syllable) |
| DICOM | DYE-com |
| Radiologist | ray-dee-OL-oh-jist |
| Trachea | TRAY-kee-uh |
| Pneumonia | nyoo-MOH-nyuh |
| Tuberculosis | too-BER-kyoo-LOH-sis |
| Genki | GEN-kee |
