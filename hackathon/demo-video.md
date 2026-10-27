# Recording a hackathon demo video

The video is a real run of the app, played by a script in a hidden Chrome
window and recorded frame by frame. Nothing is edited afterwards.

The recording tools are throwaway. Install them in a scratch folder outside
the project when the work starts. The code rules in `core/` and
`languages/` do not apply to them. Never commit them.

---

## The rules

1. **Plan with the person first.** Agree the parts, their order and every
   narration line. Ask whether the greeting should say their name.
2. **Record the real app.** Production build, real clicks, real waits.
   Visit every page once before recording, so nothing freezes.
3. **Record only the page's own sound.** Mix the narrator and the app
   inside the page and record that mix. Never the microphone, never the
   computer's sound. Headless Chrome on Windows records the whole computer
   instead of the tab.
4. **The narrator never talks over the app.** Turn the app down to about
   18 % while the narrator speaks, and speak before pressing play.
5. **Captions show the narrator's words.** One phrase at a time, centred,
   sized to the words, spoken word highlighted, timed by speech-to-text.
6. **No section labels on screen.** The narration carries the order.
7. **The camera waits, then fits.** Wait for scrolling to stop, then zoom so
   the whole element fits with padding.
8. **Highlights leave on time.** Fade the highlight at the press, during the
   camera move.
9. **Every press is visible.** Hand cursor, the button shrinks and
   brightens, a ripple, then the click.
10. **Never leave it silent.** Every click, every screen and every wait
    has the narrator saying what is happening and why, or the app's own
    voices playing. A silent stretch, even a few seconds, feels like the
    video has stopped. Fill a long wait with lines about what the app is
    doing right now.
11. **Wait for things, not for time.** Each step waits for what it needs,
    with a time limit.
12. **The recording always stops.** Save the sound in a `finally`, so a
    failed run still leaves a file.
13. **Review every take** before calling it done.

---

## What to install

Ask first. It is about 300 MB.

| Need | Use |
|---|---|
| Narrator voice | `kokoro-js`, free, runs on the computer (`am_liam` at speed 1 reads well) |
| Caption timings and review | AssemblyAI speech-to-text, with the project's API key |
| Frames | the installed Google Chrome, driven over the DevTools protocol |
| Joining frames and sound | `ffmpeg-static` |

The narrator is not an AssemblyAI voice unless the person asks for one.

---

## The parts of the video

| Part | Length | On screen |
|---|---|---|
| Greeting | 4 s | the logo and a sound wave moving with the narrator's voice |
| Intro | 15–25 s | a motion graphic of what the app does, using its real output |
| Architecture | 40–60 s | boxes that build in, dots flowing along the lines, each part lit while it is explained |
| Demo | 2–3 min | the real app: every input and every mode explained, one full run, a quick look at the others |
| End card | 8 s | name, "Built with …", the products used, tagline, repository link |

A creative app does not open with a problem.

---

## Motion graphics

The agent writes every motion graphic as code on the same stage. There is
no video editor and no stock footage.

1. **Preview first.** Show the person a page they can replay, with its
   captions, and change it until they agree.
2. **One clock.** One `render(t)` draws the whole graphic for time `t`, so
   the recording matches the preview.
3. **Time it from the sound.** Use the real lengths of the voice clips. The
   graphic waits for the voice.
4. **Real output only.** Real lines, voices, pictures and logo.
5. **Calm motion.** Ease in and out, about 1 s per move, nothing bouncing.

---

## How the recording works

1. Voice each narration line, then send each clip to speech-to-text for its
   word timings.
2. Serve a 1920 × 1080 director page and the app from one origin, with a
   small server that passes app requests through. The director can then
   read and click inside the app.
3. In the director: the app in a frame, a cursor, a camera that scales and
   moves the frame, a highlight, the motion graphics and the captions.
4. Take the app's sound from inside the app by replacing
   `AudioNode.prototype.connect` in its window, so anything sent to its
   speakers also reaches the director's mix. Record the mix with
   `MediaRecorder`.
5. Run Chrome headless at 1920 × 1080, start `Page.startScreencast`, and save
   every frame with its timestamp.
6. Join the frames, each lasting until the next, with the sound:

       ffmpeg -f concat -safe 0 -i frames.txt -i sound.webm \
         -vf "fps=30,scale=1920:1080,format=yuv420p" \
         -c:v libx264 -crf 18 -c:a aac -b:a 192k -shortest demo.mp4

A production server only serves the files it saw at start-up. Restart it
after adding files.

---

## Reviewing a take

1. Make a grid of one frame every 5 seconds:
   `ffmpeg -i demo.mp4 -vf "fps=1/5,scale=480:-1,tile=4x4" -vsync vfr sheet%02d.jpg`
2. Send the sound to speech-to-text with speaker labels and read it in
   order.
3. Check:
   - every zoom shows the whole element;
   - no highlight is left behind;
   - every line has a caption;
   - only the narrator and the app speak, never at once;
   - no gap of more than about 2 seconds without a voice;
   - every voice matches what is on screen.
4. Report what you found, with times.

---

## Do and don't

| Do | Don't |
|---|---|
| Agree the parts, lines and motion graphics first | Record before the person has agreed |
| Open with a greeting and what the app makes | Open with a "problem" for a creative app |
| Record the production build after a warm-up | Record the dev server |
| Mix and record the sound inside the page | Record the microphone, the tab or the computer |
| Turn the app down while the narrator speaks | Let the narrator and the app speak at once |
| Give every click and every wait a line or the app's voices | Leave a click, a zoom or a wait in silence |
| Show captions centred, phrase by phrase | Show one fixed line, or text pinned to one side |
| Let the narration carry the order | Put "Intro" or "Step 1" on screen |
| Fit the whole element after the scroll stops | Zoom while the page is still moving |
| Hide the highlight at the press | Leave it on something that has gone |
| Wait for an element, with a time limit | Sleep for a fixed time and hope |
| Test every input and mode before recording | Assume the untested ones work |
| Review the frames and the transcript | Call a take good without looking |
| Follow `core/git.md` for the project's commits | Commit with "Add …" messages |

---

## Mistakes already made

| Mistake | Cause | Fix |
|---|---|---|
| Zooms cut content off | zoomed while the page scrolled | wait for the scroll to settle, then fit |
| The highlight landed in the wrong place | measured before the camera stopped | measure after the camera stops |
| The highlight stayed after a click | it only left at the next camera move | hide it at the press |
| Clicks looked frozen | dev server compiling, no press shown | production build, warm-up, visible press |
| The person's voice was in the video | the microphone was mixed in | never use the microphone |
| Other sounds from the computer got in | headless tab capture recorded the whole computer | record only the page's mix |
| The video froze while recording | tab capture in headless Chrome | capture frames with `Page.startScreencast` |
| The narrator talked over the story | both played at once | duck the app, speak before play |
| Clicks, zooms and waits went by in silence | lines were written only for some steps | a line for every click and every screen, more lines for long waits |
| The narrator sounded slow or wrong | slowed speed, an AssemblyAI voice the person did not want | Kokoro at speed 1, a voice the person picked |
| The free Edge voices stopped working | they block scripted use | use Kokoro on the computer |
| Captions were easy to miss | dark, thin, one fixed line | bold, phrase by phrase, spoken word lit |
| The diagram hid the captions | captions on a lower layer | captions on the top layer |
| Captions sat to one side | full-width box, left-aligned text | centred and sized to the words |
| Fonts looked wrong | the director page never loaded them | load the fonts in the director page |
| Section labels were on screen | added as decoration | none |
| The structure was wrong | did not follow the person's order | greeting, intro, architecture, demo, end card |
| There was no greeting | missed | narrator greets over the logo |
| Only one of three modes was shown | only the main run was planned | explain and show every mode |
| A take had no sound | a step waited forever and the recording never stopped | optional steps, save the sound in a `finally` |
| A story did not start on its own | the app's autoplay did not fire | press play when the button is there |
| A story played another story's audio | an app bug | compare the transcript with the frames, fix the app |
| New files gave 404 | the production server only serves files it saw at start-up | restart it after adding files |
| The port was already in use | an old server was still running | stop the old server first |
| Pictures were missing | the image service had no credit | check credit before recording |
| An input was never tested | only "From an idea" was used | test every input before recording |
| Commits broke `core/git.md` | wrote "Add …" messages | re-read `core/git.md` before committing |

---

## Before you hand over the video

- Did the person agree the parts, every line and every motion graphic?
- Was it recorded from a production build, after a warm-up?
- Is the only sound the narrator and the app?
- Is there any stretch with no voice at all?
- Does every narrated line have a centred caption, with no section labels?
- Is every mode of the app explained and shown?
- Did you review the frame grid and the transcript?
