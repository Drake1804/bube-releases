# Connecting to the car and testing the app

**None of this has been run on a car.** The connection steps come from the documentation of the two projects they rely on, [idrive7-8-ssh-root](https://github.com/cauliflowermossgrape/idrive7-8-ssh-root) and [mgu-store](https://github.com/robertnisipeanu/mgu-store). The app's steps come from testing off the car.

## The short way: no laptop

The **YouTube Checks** app runs the tests and prints the results on the centre screen. You take photos of the screen with your phone and attach them in a Claude Code session. You need the Store on the car and some internet for it, or a USB stick.

1. **Install it.** In the Store, go to Settings, then **Add source**, and type `Drake1804/bube-releases`. Pick **Releases** if it asks. Open **YouTube Checks** and press **Install**. Without internet, use the USB stick from [step 5](#5-install-the-app).
2. **Restart the head unit** when asked, or with Settings, then **Restart head unit**. The helper that does the work may only start after this. Until then the app says `Helper not running`.
3. **Open the YouTube Checks tile** among the car's apps, parked.
4. **Press Run all checks.** It takes up to a minute and only reads.
5. **Photograph page 1, the summary.** Then press **Next page**, or turn the dial, and photograph each page. Each one says `Page 2 of 7`, so nothing gets missed. Get the whole screen in, as straight on as you can.
6. **Attach the photos** in a Claude Code session and say what you saw.

The other cards, in the order to try them:

| Card | What it does | What to tell me |
|---|---|---|
| **Video test** | Colour bars fill the screen for about 45 seconds, then the results show. This is the go or no-go for video. | Whether you saw the bars, and a photo of the results |
| **Player test** | Searches YouTube and plays a video out of sight for 30 seconds. Needs the car online: see [step 4](#4-give-the-car-internet). | A photo of the results |
| **Sound test** | Select CarPlay as the source and pause the music first. Press **Test all**, or a device, for a quiet 3-second tone. If you hear one, press **I heard ... keep it**, and the YouTube app uses that device. **Parked only: the head unit may restart.** | Which device you heard, if any, and whether it restarted |
| **Sound snapshot** | Three steps, while CarPlay is idle, playing and paused. It plays nothing and stops nothing. Use it if the tones are silent. | Photos after each step |
| **Last results** | Shows the last results again. | |

Try the YouTube app itself, then run **Run all checks** again: its logs are in the report. If the head unit restarted, open the app after it comes back: the report shows the end of the boot before it, and every test tried so far.

A photo can show firmware details and the name of your phone, though the report leaves out your account's tokens and cuts signed links, phone addresses and VIN-shaped words. That is fine to send in a session, but do not post photos publicly.

**When you are done,** remove **YouTube Checks** and **YouTube Checks helper** in the Store: the helper runs as root and has no use otherwise.

## The long way: a laptop on SSH

From a laptop that has never reached the car to a video playing, then sound, then sending logs. Written for a Mac. If a step does not do what it says here, stop and [send the logs](#8-send-logs).

Run every command on the Mac, from this repository's folder. [CAR_TEST.md](CAR_TEST.md) is the full checklist for a test session, and goes further than this page.

**Never do these:**

- Do not run the root tool's `flash` or `fix-key` again. SSH stays on without them, and a flash wipes the head unit's coding and BMW Digital Key pairings.
- Do not stop, restart or signal `carplay`. A public project reports that doing so power-cycles the head unit.
- Do not test any of this while driving.

### What you need

- **Root SSH on the head unit, already set up.** The root tool does this once per head unit.
- **The Store, already installed.** mgu-store's README has the one command that installs it.
- **Root's private key.** See step 1.
- **An ENET cable** (OBD at one end, RJ45 at the other), and a way into the Mac: an Ethernet port or a USB-C Ethernet adapter.
- **Internet for the car**, for example your phone's hotspot. See step 4.
- **Stable power.** Ignition on is enough for a short session. For anything longer, use a battery charger, or run the engine outdoors, never in a closed garage.

Keep the car key and the cable in the car for the whole project: losing either leaves no recovery path.

## 1. Set up the Mac (once)

Copy root's private key to where the rest of this page expects it, using its real path. The root tool wrote it to `data/enrollments/<VIN>/id_rsa` in its own folder. Unless you chose its "unique key" option, the same key is the tool's `data/assets/05_SSH_ROOT_KEY.pem`.

```bash
cp /path/to/id_rsa ~/.ssh/mgu_root_key
```

```bash
chmod 600 ~/.ssh/mgu_root_key
```

Give the car a short name in SSH. Current OpenSSH leaves out the older RSA signatures the head unit uses, so a plain `ssh root@...` is likely to fail without these lines. The head unit's host key can change, so it is not remembered. The address is a placeholder until step 2.

```bash
cat >> ~/.ssh/config <<'EOF'

Host bmw
    HostName 169.254.36.47
    User root
    IdentityFile ~/.ssh/mgu_root_key
    IdentitiesOnly yes
    BatchMode yes
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
    ConnectTimeout 15
    ServerAliveInterval 10
EOF
```

Download mgu-store's installer. It can find the head unit's address for you, and needs Node 18 or newer:

```bash
curl -fsSL https://raw.githubusercontent.com/robertnisipeanu/mgu-store/main/deploy/mgu-install.js -o ~/mgu-install.js
```

## 2. Connect (every session)

1. **Turn the ignition on.** With the key in the car, press Start/Stop once without touching the brake. Wait for the centre screen to finish starting.
2. **Plug in the cable.** The OBD port is under the dashboard in the driver's footwell; the handbook shows the exact place. Push it fully home, and put the RJ45 end into the Mac's Ethernet port or adapter.
3. **Set the adapter.** In System Settings, open Network, then the adapter, then Details, then TCP/IP, and set Configure IPv4 to **Using DHCP**. The car has no DHCP server, so within a minute or so the Mac gives itself a `169.254.x.x` address.
4. **Check the Mac has its address.** This should print exactly one line:

   ```bash
   ifconfig | grep 'inet 169.254'
   ```

5. **Ask for the head unit's address.** This command only lists the GitHub tokens the Store holds, so it changes nothing:

   ```bash
   node ~/mgu-install.js token --list --identity ~/.ssh/mgu_root_key
   ```

   The line you want comes first. The address in yours will differ:

   ```text
   connecting over ENET on en7 (head unit 169.254.36.47)
   ```

6. **Put that address in `HostName`,** in the `bmw` block:

   ```bash
   open -e ~/.ssh/config
   ```

7. **Log in.** This should print `uid=0(root)`:

   ```bash
   ssh bmw id
   ```

8. **See what is installed:**

   ```bash
   ssh bmw sh /var/opt/mgu-deploy/mgu-deploy.sh status --json
   ```

The address can change between sessions. If `ssh bmw` times out next time, repeat steps 5 and 6.

**Without the cable.** mgu-store also reaches the head unit over the car's own Wi-Fi hotspot, at `172.16.222.97` by default. If step 5 says `connecting over the car's hotspot`, your Mac is already on it, and you can use that as `HostName`. The root tool's README says SSH needs the cable, so this may not answer on your car: rely on the cable.

## 3. Check that video can work

Do this first. If it fails, nothing else on this page can work.

```bash
ssh bmw 'cat > /tmp/plumbing.js' < tools/plumbing/plumbing.js
```

```bash
ssh bmw node /tmp/plumbing.js bars --seconds 120
```

- [ ] Colour bars appear on the centre screen, at the right shape, and stay for the two minutes.
- [ ] Open a popup in the car's menus: the bars hide, and come back after it closes.

```bash
ssh bmw node /tmp/plumbing.js rate --seconds 60
```

- [ ] Note the number in the `stats N fps` line. Under 28 means video will not be smooth on this unit.

If you get an `error` line or no bars, read the renderer's log, then [send the logs](#8-send-logs) using the "if the tool is not installed" commands:

```bash
ssh bmw journalctl -u mgu-render --no-pager -n 50
```

## 4. Give the car internet

In the Store, go to Settings, then **Wi-Fi**, and join your phone's hotspot. The Store shows `Online` once it is connected.

mgu-store also supports an iPhone tethered over USB, but its USB music stops while it tethers, so use Wi-Fi for the sound test. Whether your iPhone can run a hotspot during wireless CarPlay is untested here. If CarPlay drops, connect CarPlay with a USB cable and keep the hotspot on Wi-Fi.

## 5. Install the app

**From GitHub**, with the car online:

1. In the Store, go to Settings, then **Add source**, and type `Drake1804/bube-releases`. Pick **Releases** if it asks.
2. On the Store page, open **YouTube** and press **Install**. This also brings the player tool and the renderer.
3. Restart when asked, or use Settings, then **Restart head unit**.

**From a USB stick**, with no internet at the car:

1. Download `mgu-repo-usb.zip` from the [release page](https://github.com/Drake1804/bube-releases/releases/tag/catalog), or with this command:

   ```bash
   gh release download catalog --repo Drake1804/bube-releases --pattern mgu-repo-usb.zip --dir ~/Downloads
   ```

2. Unzip it onto a FAT or exFAT stick, so a folder named `mgu-repo` sits at the top of the stick.
3. Plug the stick into the car, open the Store's **USB** page, install **YouTube**, and restart when asked.

Once it has restarted, reconnect as in step 2 and check the player is running. This should print `active`:

```bash
ssh bmw systemctl is-active drake1804-ytplay
```

## 6. Watch a video

Stay parked. Open the **YouTube** tile among the car's apps, search, and pick a video. The first search may be slow.

- **Touch** works everywhere.
- **The iDrive controller:** on a list, the dial moves and a push opens. On the player, the dial seeks and a push pauses. BACK goes back and stops the video.

What to expect from this build:

- Sign-in works from the QR code in the app, with nothing else to install. It lasts 7 days, then you scan again, and only accounts listed on the Google project can use it. Search and playback do not use it, but if YouTube refuses a video as a bot, a signed-in car also fetches the video cookies you set on the relay, and tries again. Those stay on the car when a sign-in expires, so video goes on until they stop working. Signing out removes them.
- Titles in anything but Latin letters show `?`.
- There is no sound until you turn it on in step 7.
- Nothing stops video while the car is moving, so that stays your job.

To see the player's numbers while a video plays:

```bash
ssh bmw "node -e 'var s=require(\"net\").connect(process.argv[1]),b=\"\";s.on(\"connect\",function(){s.write(\"status\n\")});s.on(\"data\",function(d){b+=d;if(b.indexOf(\"\n\")>=0){process.stdout.write(b);process.exit(0)}})' /run/drake1804-ytplay/ctl.sock"
```

For a full test session, follow [CAR_TEST.md](CAR_TEST.md) step 6.

## 7. Try the sound

Sound is a best guess that nobody has heard on a car. Do this parked. **The head unit may restart.** That is one of the things this step is meant to find out, and the logs survive it.

If player 0.1.7 or older has tried sound since the last restart, restart the head unit first. It may have left files that keep CarPlay out of its own sound devices.

Connect the phone to CarPlay, select CarPlay as the source, pause the music, wait 20 seconds, and set the volume to where you normally listen.

**See which sound devices the unit offers:**

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio list
```

Each line says `try`, `maybe` or `leave`, then a device name. `no sound device found` means this route has nothing to work with: go to step 8.

**Play a quiet tone on each device, 3 seconds each.** It prints each name as it plays, so note which ones you hear:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio scan
```

Heard nothing? Try once more, a little louder. Leave the volume knob alone: this path may ignore it.

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio scan --db -20
```

**Play 10 seconds on the one you heard.** `smartx:media` is only an example, so use the name it printed:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio tone --device smartx:media
```

- [ ] You hear the tone at the speakers for all 10 seconds.
- [ ] The head unit stays up, and CarPlay music and the radio still sound as they did.

**Save that device,** so the player does not have to guess:

```bash
ssh bmw /var/opt/drake1804-ytplay/current/ytaudio save smartx:media
```

**Now in the app.** Open a video and tap **Sound** on the player until it says `quiet`. The button is touch only, and it cycles off, quiet, medium, loud.

- [ ] Beside the button it says `On <device>`, and you hear the video.
- [ ] Sound and picture stay together for 5 minutes. The `status` command in step 6 shows `audioDrift` under 0.1.
- [ ] Pause stops the sound. Resume and seek bring it back, in step.
- [ ] Pause the video and start music in CarPlay: it plays, and CarPlay does not restart. Then try the other way round.
- [ ] Reverse gear and an incoming call pause the video and its sound, and you hear the car's own sounds.
- [ ] Turn the volume knob and note whether it changes the video's sound. If it does not, keep Sound at `quiet`.

Two things it cannot warn you about:

- **Silence with no error.** A device can take the sound with nothing connecting it to the speakers. The page says `On <device>` and you hear nothing.
- **A restart.** Holding the device while `carplay` wants it may restart the head unit. Nobody knows yet.

If it fails, turn Sound off, and the video keeps working. Then send the logs.

## 8. Send logs

Do this whenever something goes wrong, as soon as you can. Replace the words in quotes with what you saw and about when. Use plain words with no apostrophes, because the head unit's shell reads them too.

```bash
ssh bmw sh /var/opt/drake1804-ytplay/current/collect-logs.sh "no sound at 10:40, the page said busy" > ~/bube-logs.tgz
```

**Or send them from the car, with no laptop.** In the YouTube app, open Settings and press **Turn on** beside **Send logs online**. Write down the ID it shows, and say it along with what you saw. The app's and the player's logs then go online for 24 hours, and catch up when the car is next online, so a restart or a dead hotspot loses nothing. Press **Turn off** when you are done. This leaves out the journal, so if the head unit itself misbehaved, use the command above as well.

**If the tool is not installed,** copy the collector over first:

```bash
ssh bmw 'cat > /tmp/collect-logs.sh' < native/ytplay/src/collect-logs.sh
```

```bash
ssh bmw 'cat > /tmp/probe.sh' < tools/audio-research/probe.sh
```

```bash
ssh bmw sh /tmp/collect-logs.sh "the install failed" > ~/bube-logs.tgz
```

It takes about half a minute, and only reads. The logs on the unit survive a restart, so if the head unit restarted, wait for it to come back and run this then. If the address changed, repeat step 2 first.

Then attach `~/bube-logs.tgz` in a Claude Code session and say what you saw. Keep the file private and out of this public repository. It leaves out your account's tokens and cuts signed links, phone addresses and VIN-shaped words, but it still holds firmware text.

## 9. Undo it

- **Sound only:** tap **Sound** until it says off.
- **Everything:** in the Store, open **Installed**, then remove **YouTube** and **YouTube player**. Afterwards this should list neither:

  ```bash
  ssh bmw sh /var/opt/mgu-deploy/mgu-deploy.sh status --json
  ```

- **A stuck head unit:** use the Store's Settings, then **Restart head unit**. If the screen does not respond, hold the volume and power knob for 20 to 30 seconds until the screen goes dark, as the root tool's README describes.

## If it does not connect

| What you see | What to do |
|---|---|
| `ifconfig` prints nothing | Check the ignition is on and the cable is pushed fully home at both ends. See whether System Settings shows the adapter as connected. bootmod3's support page warns that generic USB Ethernet adapters often fail on Macs. |
| `ifconfig` prints two or more lines | Another port also has a link-local address, so traffic may go the wrong way. Unplug it, or turn off that service in Network, such as Thunderbolt Bridge. |
| `no head unit answered SSH` | The car may be asleep: press Start/Stop once and wait for the screen. Then read the next row. |
| Still nothing, or `ssh bmw` times out | The address may have changed: repeat steps 2.5 and 2.6. If it still fails, the root tool's `python ssh_enable.py status` reads the unit's address and state and changes nothing. |
| Port 22 is closed, and SSH used to work | The root tool's `python ssh_enable.py enable` re-opens it without flashing. Use it only if SSH no longer logs in. |
| `No route to host` | macOS can block apps from the local network. In System Settings, open Privacy & Security, then Local Network, and allow the app your terminal runs in. |
| `Permission denied (publickey)` | The key is wrong. Check the path in step 1, and use the key from your own run of the root tool. If you have the right key and it is still refused, stop there: the root tool's fix for this is a re-flash, which wipes the head unit's coding. |
| `no mutual signature algorithm` or `no matching host key type` | Use `ssh bmw`, so the options from step 1 apply. |
| `UNPROTECTED PRIVATE KEY FILE` | Run the `chmod` from step 1 again. |
| `Bad configuration option` or `Bad owner or permissions` | Check `~/.ssh/config`: the block from step 1 should be whole, and the file should not be writable by others. |
| The Store does not list the stick | `mgu-repo` must be at the top of the stick, not inside another folder. Check the stick is FAT or exFAT, and look on the Store's **USB** page. |

## Where this comes from

Read on 3 November 2026. Please trust the projects' own pages over this one if they have changed.

- [idrive7-8-ssh-root](https://github.com/cauliflowermossgrape/idrive7-8-ssh-root) at `e07f0be`: the one-time setup, the key's location, the link-local address, the warning about flashing, and the forced restart.
- [mgu-store's README](https://github.com/robertnisipeanu/mgu-store) and [installer](https://github.com/robertnisipeanu/mgu-store/blob/main/deploy/mgu-install.js) at `d2ea71b`: finding the head unit, the hotspot address, the SSH options, sources, USB sticks and internet.
- [bootmod3's laptop instructions](https://bootmod3.atlassian.net/wiki/spaces/BCS/pages/463445): the warning about adapters on a Mac. This is a vendor's support page.
- [BimmerCode with an ENET cable](https://blog.obdii365.com/2022/09/07/how-to-set-up-bimmercode-with-bmw-enet-cable/): the port in the driver's footwell. This is a seller's blog, so check your handbook.
