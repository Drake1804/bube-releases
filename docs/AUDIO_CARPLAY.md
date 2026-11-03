# Sound through the factory CarPlay path

Can this project reach the speakers the way the factory CarPlay service does, with no iPhone playing the audio? Not known yet. The code is written on a best guess, and has never run on a car.

**Nothing below has been seen on a car by this project.** Every statement about the head unit is someone else's report, a document about other hardware, or an inference, and is marked as such. Sources are at the end, read on 2026-11-03.

**The plan's gate was moved.** It said audio goes into the app only after a demonstrated 10-second tone from a root shell to the speakers. On 2026-11-03 the owner chose to build first, on a best guess. So the code exists, but it stays off until someone turns it on, and the tone is now the first thing to try on the car. See [The tone first](#the-tone-first).

## In short

The unit's AVB stack looks like Intel's IAS AVB Stream Handler, or something built from it. In Intel's public version, apps do not touch AVB: they write to an ordinary-looking ALSA device, `smartx:<name>` or `avb:<name>`, served by a userspace plugin over shared memory, and the handler turns that into AVB.

If the car works that way, "AVB, not ALSA" (this project's working assumption so far) is wrong at the layer an app sees, and a `carplay` process may simply be an ALSA client. It would also explain why no public project reports a path: the kernel's `/proc/asound` would show nothing while music plays. **All of this is unverified.** The probe, `tools/audio-research/probe.sh`, settles it in about five minutes without changing anything. [docs/CAR_TEST.md, step 7](CAR_TEST.md) says how to run it.

Even if the device is there, the likely obstacle is routing: Intel's device only makes sound when something has connected it to the stream going out, and that call is not one this project can make yet.

## What was built on that guess

Route A below. [native/ytplay/README.md](../native/ytplay/README.md#sound) has how it works, and [CAR_TEST.md step 8](CAR_TEST.md) the order to try it in.

| Unverified assumption | What the code does about it | If it is wrong |
|---|---|---|
| The firmware has the SmartX plugin, or a named ALSA device | Finds devices at run time from `/run/smartx`, `/dev/shm` and ALSA's configuration, instead of naming one | The page says there is no device; `ytaudio list` shows what is there |
| The files are named as in Intel's public code (V5) | `smartx_<name>_p` becomes `smartx:<name>`; also `avb_` | Nothing is found from the files; ALSA's configuration may still list them |
| A name says what a device carries | Opens only `carplay`, `media`, `entertain`, `music`, `stereo`, `main` by itself, and never names with `phone`, `call`, `pdc`, `gong`, `warn`, `mic` | It may pick a wrong one, or none. `ytaudio scan` and `ytaudio save` settle it by ear |
| 48 kHz, 16-bit stereo is accepted (V2) | Tries that, then the same through ALSA's `plug` converter | The page says why it would not open |
| uid 3920 may use it | Keeps the groups that own the plugin's files at the player's start | "Permission denied" on the page and in the log |
| Something has connected the device onward (V2, V6) | **Nothing: this project cannot do that.** CarPlay selected and paused is the best chance | It opens and takes samples, and nothing is heard. No error will say so |

What it does to limit harm, none of it tried on a car:

- Off by default, and quiet when turned on: -30 dB, because a device past BMW's volume control may ignore the volume knob.
- The device is held only while sound is written, and let go at a pause, a stall or a popup.
- It runs in its own process with the crash handler told about it, so a crash in BMW's plugin should cost the sound and not the picture or the unit (K5). The video was tested to play on through a crash, a hang, a busy device and no device.
- Logs reach the disk before each device is opened, so a restart of the unit leaves a trace.

Risks that remain, and why to try this parked:

- **`carplay` may need the device while this holds it.** It would get `-EBUSY` (V5). Nobody knows what it does then. A failed end of CarPlay restarts the head unit (K4).
- **A crash in the Stream Handler** restarts the head unit if it is not on the lists (K5, K6), and may take the reverse camera with it until then (I8).
- **It might sound over a warning or a call.** The app pauses when the HMI takes over the screen, but a chime with no popup would not pause it, and it may not be mixed as the car's own sources are.

## What is known

### Reported from real head units by public projects

Not checked by this project. Most comes from one G30 on one build.

| # | Report | Source |
|---|---|---|
| K1 | `carplay.service` runs `/usr/bin/carplay -c /etc/cinemo/cinemo_options.xml`, with `OnFailure=mediaddpserver_kill.service`. | [multitouch `agent.js`][mt-agent] refuses a unit without both |
| K2 | The patched build is 3,484,912 bytes, SHA-256 `ddb33e33…3e50`, software `P18_23w44.2-1-3`, on one G30, with an Android AI box. Direct iPhone is untested, and any other hash is refused. | [multitouch README][mt] |
| K3 | The same hash is called carplay 22.28.3 by a second project. | [waze-on-cluster README][waze] |
| K4 | CarPlay crashes on SIGTERM, and `systemctl stop` or `restart` of it power-cycles the head unit. Any failed end of a CarPlay run sends `mediaddpserver` SIGQUIT, whose core dump restarts the unit. | [multitouch README][mt], [PATCH.md "Activation"][waze-patch] |
| K5 | A core dump goes to the crash handler, which reboots the unit unless the executable's name is in `/etc/crashhandler-restartlist.conf`. `node` is listed. The kernel pipes no dump for a process whose core limit is 1 byte. | [mgu-store deploy README][deploy] |
| K6 | A stopped service that recmgr does not list reboots the unit. | [deploy README][deploy], [issue 27][i27] for iDrive 8 |
| K7 | CarPlay's second video stream goes to a Cinemo sink URL held as a string in the binary: `avb://media_transport.avb_streaming.15?mime_type=video/H264`. A capture on a car showed RTP. A `file://` sink is rejected. | [PATCH.md "Edit 2"][waze-patch] |
| K8 | `/usr/lib/libias-media_transport-avb_ufipc_bridge.so.1.3.10` exports senders and receivers for H.264 and MPEG-TS packets (`ias_avbbridge_*`). Its export list has no audio call. | [`libias_avb.symbols`][waze-abi] |
| K9 | The bridge's shared memory is `sin_ufipc:mgu_ufipc`, mode 0660. Root could join stream 15; `hmi_5000` could not until given the group. | [waze service unit][waze-unit] |
| K10 | The unit has `avbstreamhandler.service` and `avbsourcesinkcontroller.service`. The CarPlay-to-cluster talker is named `AVB_MGU_CarPlay_Kombi`, stream `0x301a9ee108e`. | [`patchd.js`][waze-patchd] |
| K11 | Audio and video to the rest of the car run over AVB through `igb_avb`, outside the normal network stack. `mediaengine` and `mediaapp` are the media services. Camera views also arrive as H.264 over AVB. | [idrive7-apps README, 4.2 and 4.6][apps] |
| K12 | `/etc/oap/someip/someip-interfaces.json` is the SOME/IP catalog, read as `interfaces`, `datatypes` and `instanceIDs`. Harman's name is on some of its services. | [idrive7-apps README, 4.2 and 4.5][apps], [`sigdoc.py`][apps-sigdoc] |
| K13 | A stock media daemon holds a connected iPhone for iAP. Tethering over USB takes the phone from it, and USB media stops. | [mgu-uplink README][uplink] |

The other two sources named for this work add nothing on sound. [IterioDev/car-apps][iterio] names no audio in its README or file list; its packages were not unpacked. [Issue 27][i27] holds the owner's question from 2026-11-03, with no reply when read.

### BMW service bulletins

| # | Report | Source |
|---|---|---|
| K14 | On a list of models that includes G05, a failed Receiver Audio Module (RAM) means no telephony, navigation or radio audio, and system sounds such as PDC and turn indicators may fail too. The check named is the Ethernet link between head unit and RAM. | [SIB 65 16 23][sib-2023] |
| K15 | On G05 and others, a RAM software error left no audio for about 30 seconds after head unit start, with a RAM fault code stored. | [SIB 65 22 19][sib-2019] |

### Intel's public documents, about Intel's reference hardware and not this car

Both projects are archived. The firmware's copy may differ in any detail.

| # | Statement | Source |
|---|---|---|
| V1 | The Stream Handler links an AVB Ethernet interface to local apps: audio through ALSA, video as H.264 or MPEG-TS. It uses `igb_avb` and `/dev/igb_avb`. | [README][ias] |
| V2 | Each local stream creates an ALSA device, usable with `aplay`. A local stream carries data only once connected to a network stream. Audio is 16-bit at 48 or 24 kHz, and the device does not fully emulate a sound card. | [ICD][ias-icd] |
| V3 | The example plays to an external amplifier with `aplay -Davb:stereo_0`, and warns against doing it from a serial console, whose driver can preempt the real-time threads. | [example systems][ias-ex] |
| V4 | `pcm.avb` is an alias of `pcm.smartx`, whose plugin is `libasound_module_pcm_smartx.so`. It lives in `/usr/share/alsa/alsa.conf.d/` or `/etc/alsa/conf.d/`. Its files are in `/run/smartx`, group `ias_audio`. | [config template][ias-conf], [ICD][ias-icd] |
| V5 | The plugin only joins a connection the daemon already made, else `-ENODEV`. One opener at a time: a second gets `-EBUSY`, through a lock such as `/run/smartx/smartx_<name>_p.lock`. The player's sound helper relies on this naming to find devices. | [`IasAlsaSmartXConnector.cpp`][am-conn], [open-once test][sx-once] |
| V6 | SmartXbar routes with `connect(source, sink)` through an in-process API. A sink port takes one connection. | [`IasRoutingImpl.cpp`][sx-route] |
| V7 | Test tones can be generated inside the Stream Handler, but are set up by its configuration library. | [ICD][ias-icd] |

### Another car, as an analogy only

One person's binary analysis of GM Info 3.7, also Apollo Lake, but Android: `libNmeCarPlay.so` from Cinemo, `libias-audio-smartx.so`, `libias-media_transport-avb_streamhandler.so`, and Harman libraries, with a PulseAudio layer between. It shows the same parts on the same chip family and proves nothing about BMW. [Source][gm]. The `Nme` prefix also fits `NmeVfsSocket` in [PATCH.md][waze-patch].

## What is only inferred

| # | Inference | Why | Against | How firm |
|---|---|---|---|---|
| I1 | The unit's AVB stack derives from Intel's IAS code. | K8, K10, K11 against V1: the `Ias` namespaces, the `media_transport` name, `igb_avb`, the H.264 and MPEG-TS split | Exported names differ: `ias_avbbridge_*` on the car, [`ias_avbvideobridge_*`][ias-bridge] in public. The UFIPC bridge is not in the public code | Fairly firm |
| I2 | Head unit audio does not pass through that bridge. | K8 has no audio call, and V1 sends audio through ALSA | An export list is not a call graph | Medium |
| I3 | `carplay` writes audio to an ALSA device, probably the SmartX plugin. | I1, I2 | Never observed. It could use another Cinemo sink, hand audio to `mediaengine`, or use a layer like GM's | Medium to low |
| I4 | `/proc/asound` may look idle during CarPlay music. | V4: the plugin is userspace, so the kernel sees no stream | Only if I3 holds | Firm if I3 holds |
| I5 | Head unit audio reaches the RAM as AVB, and every sound goes through the RAM. | K11, K14 | K14 says Ethernet, not AVB | Medium |
| I6 | Routing is decided outside `carplay`. `avbsourcesinkcontroller` is a candidate. | V6, and the name in K10 | Only the name | Low |
| I7 | An ALSA route is not locked to one build the way a patch is, though device names may differ between firmwares. | K2 and K7 are byte offsets and hashes; a device name is configuration | Unknown until two builds are compared | Medium |
| I8 | Disturbing the Stream Handler could take the reverse camera with it until a restart. | K11 | Not reported by anyone | Treat as true |

## What could not be established

- Which device, if any, `carplay` writes sound to. No public code or write-up says, and Cinemo's documentation is not public.
- Whether the firmware has the SmartX plugin or `/run/smartx` at all.
- What connects a source to the stream going to the RAM, and whether an outside process can ask it to.
- Whether a device can be opened while CarPlay is connected but paused.
- Whether the catalog offers a way to play a file or URL, or to request an audio source. It is firmware-derived and not public, which is why the probe reads it on the car.
- Whether the owner's build matches K2. Everything in K1 to K10 may differ on a 2019 X5.
- Anything in DLT. Intel's code logs there, and the journal may be silent about audio. The probe does not touch DLT, as [one project warns][waze] that it is shared across the unit.

## Routes

Ranked by how likely and how risky together. None is proven.

| | Route | How likely | Risk to the car | Tied to one build | Phone |
|---|---|---|---|---|---|
| **A** | Write to the ALSA device | Best lead | Medium | No patch; names may vary | Perhaps connected and silent |
| **B** | Ask BMW's media service to play | Possible | Low to medium | Interface may vary | No |
| **C** | Add a stream through the Stream Handler | Unlikely | High | Yes, in practice | No |
| **D** | Patch or wrap `carplay` | Possible, poor fit | High | Yes, exact hash | Yes, in a session |
| **E** | Send our own AVB packets | Rejected | Highest | No | No |

**The owner's idea is route A.** It asks what `carplay` opens, then opens the same thing. Route D is the same idea done by changing `carplay`.

### A. Write to the ALSA device

Play 16-bit PCM into `smartx:<name>` or `avb:<name>`, using the unit's own `libasound`. Sound is only heard if that device is connected onward, so the likely form is the entertainment device `carplay` uses, while CarPlay is the selected source and paused. That needs a connected phone but plays nothing from it.

- **Proved by:** the probe showing `carplay` or `mediaengine` mapping `libasound_module_pcm_smartx.so`, holding a `/run/smartx/*.lock`, or mapping `/dev/shm/smartx*`. Then, past the probe, the tone.
- **Killed by:** no such plugin in `static/file-names.txt`, and no audio process mapping `libasound`.
- **Weakened by:** a device held open even while paused (`-EBUSY`, V5), or one that takes samples and stays silent because nothing connected it (V2, V6). Either pushes toward B.
- **Risk:** a bad client beside a real-time process. A crash of the handler restarts the head unit if it is not on the lists (K5, K6) and may take the reverse camera with it (I8). The level could be loud. Parked only. A restart is the recovery; nothing is written to the unit.

### B. Ask BMW's media service

Have `mediaengine` or `mediaapp` play a file or stream, or request an audio source, over SOME/IP. BMW's own code would then handle routing, volume and interruptions. It may also pair with A: B opens the route, A feeds it. Lip sync would be harder, since the service owns the clock.

- **Proved by:** `static/catalog-audio.json` holding a method that takes a file, URL or source, on an interface a local client may call. Then one call, reviewed first.
- **Killed by:** only transport controls for devices the unit has indexed, and no way to request a source.
- **Risk:** a wrong call could crash the service, with the restart in K6. vsomeip policy appears to be per uid ([deploy README][deploy]). It may change the source shown on the HMI.

### C. Add a stream through the Stream Handler

Create a local stream and connect it (V2), or use the built-in tone (V7). In Intel's design this is done at startup by a configuration library, so it means replacing a firmware file and restarting `avbstreamhandler`, which reboots the unit unless listed (K6).

- **Proved by:** a control interface in the catalog or in `static/file-names.txt`, for example a D-Bus policy file, reachable at run time.
- **Killed by:** no such interface.
- **Risk:** high. A mistake affects all head unit audio and possibly cameras (I8).

### D. Patch or wrap `carplay`

The public projects show the method works on one build, for video and touch (K2, K7). For sound it needs a live CarPlay session, because the process has nothing to play without one, and it could only replace what the phone sends.

- **Proved by:** `static/strings-carplay.txt` showing an audio sink beside the video one, and `static/hashes.txt` matching K2, or a port to the owner's build.
- **Killed by:** route A working, which makes this pointless. Or a hash that differs, unless the owner wants a port, which [the multitouch project][mt-contrib] treats as separate analysis for each build.
- **Risk:** high. A failed CarPlay end restarts the head unit (K4), and [one project][waze-patch] documents a boot loop as the failure it had to guard against.

### E. Send our own AVB packets

Build IEEE 1722 packets and put them on the wire. Rejected. In Intel's design the Stream Handler owns the hardware queues (V1), the RAM would presumably expect known stream IDs, and a RAM that fails can silence PDC and turn indicator sounds and store fault codes (K14, K15). Whether stray packets could do that is unknown, and this car is the wrong place to find out. Only worth reopening on a bench with a spare RAM.

## Reading the probe's results

Start with `compare.txt`, then `playing/summary.txt`.

| Seen | Means |
|---|---|
| `carplay` maps the SmartX plugin or holds `/run/smartx/*.lock` only in `playing` | A is real, and the device is free when paused |
| The same in `idle` and `after` too | A is real, but the device is held: expect `-EBUSY` |
| A device name in `static/config.txt` or `static/strings-carplay.txt` | The name to use in a tone plan |
| `/proc/asound` shows `RUNNING` and a moving `hw_ptr` in `playing` | Plain ALSA: simpler than expected |
| `/proc/asound` closed throughout, and no plugin anywhere | Audio leaves `carplay` some other way: look at its top threads in `activity.txt`, its sockets, and `who-maps.txt` |
| `mediaengine` grows in `activity.txt` when `carplay` plays | CarPlay may hand audio to it: weighs toward B |
| `missing.txt` names a service | A different name on this firmware; look in `processes.txt` |
| `static/identity.txt` says it is not the known build | K1 to K10 may not apply, and D means a port |
| The app's page says `On <device>`, and nothing is heard | The device took the samples and nothing connected it onward: A's main way to fail. Try with CarPlay selected and paused, then route B |

The probe was checked on a fake head unit (`tools/audio-research/test_probe.sh`), and on a real Linux `/proc` under BusyBox and Node 8.16.2, in a container with a read-only root. Its behaviour on a real head unit is unverified.

## The tone first

The app's Sound button should stay off until this has passed. Parked, on stable power, over SSH and not serial (V3), volume down first.

1. **Probe,** if you can. Read-only. `collect-logs.sh` includes a snapshot, or [CAR_TEST.md step 7](CAR_TEST.md) takes a fuller one.
2. **`ytaudio list`, then `ytaudio scan`.** A low tone on each device in turn, saying which. What crash and restart would mean is in the unit's own lists, in the probe's `static/config.txt`.
3. **Demonstration.** `ytaudio tone --device <what you heard>`: ten seconds, from a root shell, heard at the speakers. The unit stays up, CarPlay and radio sound normal afterwards, and no new RAM fault is stored.
4. **`ytaudio save <that device>`,** and only then the Sound button.

If nothing is heard on any device, the code is doing what it can and the guess about routing is wrong: send the logs, and route B is next.

## Sources

- **Multitouch patch:** [README][mt], [`agent.js`][mt-agent], [CONTRIBUTING][mt-contrib]
- **idrive7-apps:** [README, section 4][apps], [`sigdoc.py`][apps-sigdoc]
- **mgu-store:** [mgu-uplink README][uplink], [deploy README][deploy], [issue 27][i27]
- **IterioDev:** [car-apps][iterio]
- **waze-on-cluster:** [README][waze], [PATCH.md][waze-patch], [`libias_avb.symbols`][waze-abi], [service unit][waze-unit], [`patchd.js`][waze-patchd]
- **BMW bulletins, as filed with NHTSA:** [SIB 65 16 23][sib-2023], [SIB 65 22 19][sib-2019]
- **Intel AVB Stream Handler:** [README][ias], [ICD][ias-icd], [example systems][ias-ex], [config template][ias-conf], [video bridge header][ias-bridge]
- **Intel audio code:** [`IasAlsaSmartXConnector.cpp`][am-conn], [open-once test][sx-once], [`IasRoutingImpl.cpp`][sx-route]
- **GM Info 3.7, as an analogy:** [`intel_audio.md`][gm]

[mt]: https://github.com/robertnisipeanu/idrive7-carplay-multitouch/blob/ab1d07e808f79f17da7d2ec88fece1e116c523ad/README.md
[mt-agent]: https://github.com/robertnisipeanu/idrive7-carplay-multitouch/blob/ab1d07e808f79f17da7d2ec88fece1e116c523ad/agent.js
[mt-contrib]: https://github.com/robertnisipeanu/idrive7-carplay-multitouch/blob/ab1d07e808f79f17da7d2ec88fece1e116c523ad/CONTRIBUTING.md
[apps]: https://github.com/cauliflowermossgrape/idrive7-apps/blob/25c7167ce88bcf3a709ad973ac07ad0570ce89f6/README.md#4-how-the-head-unit-works
[apps-sigdoc]: https://github.com/cauliflowermossgrape/idrive7-apps/blob/25c7167ce88bcf3a709ad973ac07ad0570ce89f6/kit/tools/sigdoc.py
[uplink]: https://github.com/robertnisipeanu/mgu-store/blob/42457740ff2ca20f01696b23052c953362ba48f5/native/mgu-uplink/README.md
[deploy]: https://github.com/robertnisipeanu/mgu-store/blob/42457740ff2ca20f01696b23052c953362ba48f5/deploy/README.md
[i27]: https://github.com/robertnisipeanu/mgu-store/issues/27
[iterio]: https://github.com/IterioDev/car-apps/tree/2a897bfdec8afe9246904fdebc5e5ae9b9c6fdc6
[waze]: https://github.com/robertnisipeanu/idrive-apps/blob/8e978c93371d283765ed99cc9b2ca6e6af7167c1/native/waze-on-cluster/README.md
[waze-patch]: https://github.com/robertnisipeanu/idrive-apps/blob/8e978c93371d283765ed99cc9b2ca6e6af7167c1/native/waze-on-cluster/patch/PATCH.md
[waze-abi]: https://github.com/robertnisipeanu/idrive-apps/blob/8e978c93371d283765ed99cc9b2ca6e6af7167c1/native/waze-on-cluster/abi/libias_avb.symbols
[waze-unit]: https://github.com/robertnisipeanu/idrive-apps/blob/8e978c93371d283765ed99cc9b2ca6e6af7167c1/native/waze-on-cluster/systemd/waze-on-cluster.service
[waze-patchd]: https://github.com/robertnisipeanu/idrive-apps/blob/8e978c93371d283765ed99cc9b2ca6e6af7167c1/native/waze-on-cluster/patch/patchd.js
[sib-2023]: https://static.nhtsa.gov/odi/tsbs/2023/MC-10235203-9999.pdf
[sib-2019]: https://static.nhtsa.gov/odi/tsbs/2019/MC-10169474-9999.pdf
[ias]: https://github.com/intel/AVBStreamHandler/blob/45558f68e84cc85fa38c8314a513b876090943bd/README.md
[ias-icd]: https://github.com/intel/AVBStreamHandler/blob/45558f68e84cc85fa38c8314a513b876090943bd/public/doc/01_icd.md
[ias-ex]: https://github.com/intel/AVBStreamHandler/blob/45558f68e84cc85fa38c8314a513b876090943bd/public/doc/02_example_systems.md
[ias-conf]: https://github.com/intel/AVBStreamHandler/blob/45558f68e84cc85fa38c8314a513b876090943bd/public/res/51-smartx.conf.in
[ias-bridge]: https://github.com/intel/AVBStreamHandler/blob/45558f68e84cc85fa38c8314a513b876090943bd/public/inc/media_transport/avb_video_bridge/IasAvbVideoBridge.h
[am-conn]: https://github.com/intel/AVB-AudioModules/blob/a38fc059bb841b949bc3937593abed593c8df22e/private/src/alsa_smartx_plugin/IasAlsaSmartXConnector.cpp
[sx-once]: https://github.com/intel/SmartXbar/blob/a89ebdb37c90b69fc49f5097bfcd341824c10499/private/tst/alsa_plugin_open_once/src/alsa_plugin_open_once.cpp
[sx-route]: https://github.com/intel/SmartXbar/blob/a89ebdb37c90b69fc49f5097bfcd341824c10499/private/src/smartx/IasRoutingImpl.cpp
[gm]: https://github.com/lvalen91/gminfo_resources/blob/43316749b853bd6bfcdb14147dd29a381066e2bf/audio/intel_audio.md

All links are pinned to the commit read, except the issue and the two bulletins. Code from these repositories was read and never run. This repository copies no BMW file, and the probe's output stays out of it.
