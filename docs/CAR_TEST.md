# On the car

Run from the driveway with the engine running or on a charger, the ENET cable connected, a laptop on SSH. Keep the key and the cable in the car for the whole project: losing either leaves no recovery path.

[CAR_GUIDE.md](CAR_GUIDE.md) says how to connect and install. It also sets up `bmw`, the SSH name the commands on this page use.

## If anything here does not work

Gather the logs, as soon as you can after it happens, and bring the file to a session. Say what you saw and about when:

```bash
ssh bmw sh /var/opt/drake1804-ytplay/current/collect-logs.sh "no sound at 10:40, the page said busy" > ~/bube-logs.tgz
```

If the tool never installed, it is not on the unit yet. Copy it over first:

```bash
ssh bmw 'cat > /tmp/collect-logs.sh' < native/ytplay/src/collect-logs.sh
```

```bash
ssh bmw 'cat > /tmp/probe.sh' < tools/audio-research/probe.sh
```

```bash
ssh bmw sh /tmp/collect-logs.sh "the install failed" > ~/bube-logs.tgz
```

It takes about half a minute and changes nothing. The logs on the unit survive a restart, so a crash can be collected afterwards. Keep the file private and out of this repository: it holds firmware-derived text, though it leaves out your account's tokens and cuts signed links, phone addresses and VIN-shaped words.

## 1. Plumbing check (about 2 hours; go or no-go for the project)

```bash
ssh bmw 'cat > /tmp/plumbing.js' < tools/plumbing/plumbing.js
```

```bash
ssh bmw node /tmp/plumbing.js bars --seconds 120
```

- [ ] The bars appear over the HMI at the right aspect, and last.
- [ ] They hide when the HMI opens a popup, and come back.
- [ ] No `error` line. If there is one, or nothing is drawn: `journalctl -u mgu-render` before anything else.

```bash
ssh bmw node /tmp/plumbing.js rate --seconds 60
```

- [ ] `stats N fps`: note N. Under 28 means video will not be smooth on this unit, whatever the player does.

Please report the result on the mgu-store repository either way: its video path has never been confirmed on a car.

## 2. Things to record (open in the plan)

- [ ] `ls /usr/lib/dri /usr/lib/x86_64-linux-gnu/dri`: libva and the i915 media driver would free the CPU with hardware decode.
- [ ] MainApp version, and which Wi-Fi chip (`ls /sys/class/net`).

## 3. Rollback drill (once, before the first real install)

- [ ] Install the tool with its `ytplay_check.js` broken on purpose; `mgu-deploy` rolls it back by itself and the head unit boots clean.
- [ ] `mgu-deploy restore stock`, then diff the two MainApp bundles against the laptop's copies.

## 4. The resolver, over SSH, before the app

Over the phone hotspot, 20 test videos: music video, 4K source, livestream, age-gated, shorts.

- [ ] Each gives a 720p H.264 URL or a clean reason (`resolve.py` command in `native/ytplay/README.md`), in under 4 s. QuickJS may make the first one slow; note the time.

## 5. The player, from the shell

These two run in a shell on the unit (`ssh bmw`). If its `nc` does not know `-U`, [CAR_GUIDE.md](CAR_GUIDE.md) step 6 asks for `status` with `node` instead.

```bash
printf 'play dQw4w9WgXcQ\n' | nc -U /run/drake1804-ytplay/ctl.sock
```

```bash
printf 'status\n' | nc -U /run/drake1804-ytplay/ctl.sock
```

- [ ] A 10-minute 720p video: `dropped` under 1 % of `frames`, `top` shows ffmpeg under 200 % CPU. If not, set Quality to 480p in Settings and note it.
- [ ] `nft list table inet mgu_uplink` shows uid 3920 accepted.
- [ ] `/dev/shm/mgu-video-yt` is owned by root, and `ps` shows `ytplay` as uid 3920.

## 6. A test session with the app

Before: `mgu-deploy status --json`, `journalctl -b > before.log`, note `oapcontroller`'s crash count file.

- [ ] Install the app; restart the head unit; the tile appears; open it; search; play 10 minutes at 720p.
- [ ] Measure: `STATS` lines in `journalctl -u mgu-render` (presented fps, render time), `top` for ffmpeg and HmiMain3D, `/sys/class/thermal` for the SoC temperature at minute 10.
- [ ] Controller: the dial moves the focus on the list, the push opens; on the player the dial seeks and the push pauses; BACK goes back and stops.
- [ ] Screens: the picture sits right of the controls at the right aspect, and nothing covers it. Measure a button with a ruler: the designs assume 80 px is about 12 mm, on a 12.3-inch screen.
- [ ] Focus: note where the HMI's focus starts on each page and after BACK. The app cannot move it, so it starts on the first thing, Search, not on the first video as the designs show.
- [ ] Sign in, parked, with the private **YouTube sign-in** package installed (`native/signin/README.md`): the QR code opens on the phone, approving there brings "You're signed in", and Home, Subscriptions, Playlists and Liked fill in. Restart the head unit: still signed in. Sign out in Settings: the tabs go, and Google's account page no longer lists the app.
- [ ] `ls -la /var/opt/drake1804-ytplay/state`: the token file is 0600 and uid 3920's.
- [ ] Interrupt: incoming call, reverse gear (the camera must take the screen), HMI popup, BACK, HOME. The video hides and comes back, the position kept.
- [ ] Network loss: turn off the hotspot mid-play. The page says it is waiting, and playback resumes within 10 s of the hotspot returning.
- [ ] Crash injection: `kill -9` the player (recmgr restarts it, not the head unit); `kill -9` the app's node (MainApp restarts it, never reaching 5 restarts in 10 minutes); `kill -9 mgu-render` (the HMI stays up, the app reconnects).
- [ ] Power: ignition off mid-play, back on after 2 minutes. The unit sleeps and wakes with the app closed and nothing in `journalctl -p err`.
- [ ] Three cold starts in a row with the tool enabled: boot time within 5 s of stock (`systemd-analyze`).
- [ ] Remove both from the Store: `mgu-deploy status --json` clean, `nft` rules gone, `/dev/shm/mgu-video-yt` gone.

After: diff `journalctl -b` against `before.log` for new `err` lines; `patch_mainapp.js check --json` unchanged.

**Fail the session if:** ffmpeg over 200 % of 400; HmiMain3D frame time worse than stock while the video shows; presented rate under 28 fps for 30 fps content; the SoC over its throttle point after 10 minutes; app RSS over 100 MB.

## 7. Audio probe (about 5 minutes; any time, separate from the steps above)

Not needed for v1. It collects what [AUDIO_CARPLAY.md](AUDIO_CARPLAY.md) needs to choose a route for sound. It is read-only: it plays nothing, stops nothing, and writes only to `/tmp/bube-audio-probe`.

Parked, with the phone connected to CarPlay and nothing playing. Use SSH, not a serial console. If the unit's internet comes from the same iPhone over USB, turn that off first: it takes the phone away from the media daemon.

```bash
ssh bmw 'cat > /tmp/probe.sh' < tools/audio-research/probe.sh
```

```bash
ssh bmw sh /tmp/probe.sh idle
```

Start music in CarPlay at a low volume and wait 20 seconds.

```bash
ssh bmw sh /tmp/probe.sh playing
```

Pause the music and wait 20 seconds.

```bash
ssh bmw sh /tmp/probe.sh after
```

Copy it to a folder outside this repository:

```bash
ssh bmw 'tar czf - -C /tmp bube-audio-probe' | tar xzf - -C ~
```

Then clear it off the unit:

```bash
ssh bmw rm -r /tmp/bube-audio-probe /tmp/probe.sh
```

- [ ] `compare.txt` and `playing/summary.txt` are there and not empty.
- [ ] Note whether CarPlay was wired or wireless, and that the music was heard during `playing`.
- [ ] Bring the folder to a later session. Do not commit or post it: it holds firmware-derived text and may name your phone.

**While you are in there:** never `systemctl stop` or `restart` `carplay`, or signal it. A public project reports that doing so power-cycles the head unit. The probe needs none of that.

## 8. Sound (about 15 minutes; parked; after step 6 has shown the picture works)

Sound is a best guess that has never been heard on a car, and it is off until you turn it on. [AUDIO_CARPLAY.md](AUDIO_CARPLAY.md) says what it assumes and what could go wrong. A failed CarPlay end restarts the head unit, so expect that this may happen, and never try this while driving.

Connect the phone to CarPlay, select CarPlay as the source, pause the music, and turn the volume to where you normally listen. Then see what the unit offers:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio list
```

Play a quiet tone on each device in turn, 3 seconds each, and note which ones you hear:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio scan
```

Nothing heard: try once more with `--db -20`. Keep the volume where it is; a device past BMW's volume control may ignore the knob.

Ten seconds on the one you heard, in place of `smartx:media`. This is the plan's test, and the rest depends on it:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio tone --device smartx:media
```

- [ ] The tone is heard at the speakers for all 10 seconds.
- [ ] The head unit stays up. Afterwards, CarPlay music and the radio sound as they did before.

Keep it, so the player does not have to guess:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio save smartx:media
```

Now in the app. Open a video, and tap **Sound** on the player until it says `quiet`.

- [ ] Beside the button it says `On <device>`, and you hear the video.
- [ ] Sound and picture stay together for 5 minutes. `status` over the socket shows `audioDrift` under 0.1.
- [ ] Pause: the sound stops. Resume, and seek by 10 s: it comes back, in step.
- [ ] CarPlay: pause the video and start music on the phone. It plays, and CarPlay does not restart. Then the other way round.
- [ ] Reverse gear and an incoming call: the video and its sound pause, and the car's own sounds are heard.
- [ ] Turn the volume knob: note whether it changes the video's sound. If it does not, keep Sound at `quiet`, and say so.

Heard nothing, or it says `No sound`, or the unit restarted: that is what this step is for. Gather the logs as at the top of this page, say which of these it was, and turn Sound off.
