# YouTube for iDrive 7: releases

Packages for a personal YouTube player on BMW iDrive 7 (MGU), installed through [mgu-store](https://github.com/robertnisipeanu/mgu-store). This repository holds only the releases and their guides. Not made by or affiliated with BMW, Google or YouTube.

## Install

The car needs to be online, and you don't need a GitHub account.

1. In the Store, go to Settings, then **Add source**, and type `Drake1804/bube-releases`.
2. Pick **Releases** if the Store asks.
3. Install **YouTube Checks**, **YouTube**, or both. YouTube also brings the player tool and the renderer. Restart when asked.

The [latest release](https://github.com/Drake1804/bube-releases/releases/tag/catalog) says what works and what doesn't. The [car guide](docs/CAR_GUIDE.md) covers testing, and the [checklist](docs/CAR_TEST.md) goes further.

## Source

The project is developed in a private repository. Each release includes its full source as `bube-source-<commit>.tar.gz`, under GPL-3.0-or-later ([LICENSE](LICENSE)). The commands in the guides run from that unpacked folder. Third-party parts are listed in each package's `LICENSES.txt`.
