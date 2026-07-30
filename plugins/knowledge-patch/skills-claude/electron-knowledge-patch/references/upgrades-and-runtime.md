# Upgrades and Runtime

Use this reference to select an upgrade target, verify embedded runtimes, and
plan platform and artifact support.

## Runtime stack changes

- Electron 34 embeds Chromium `132.0.6834.83`, Node.js `20.18.1`, and V8
  `13.2` (34.0.0). Compared with Electron 33, Chromium moves from 130 to 132,
  Node.js from `20.18.0` to `20.18.1`, and V8 from 13.0 to 13.2.
- Electron 35 embeds Chromium `134.0.6998.44`, Node.js `22.14.0`, and V8
  `13.5` (35.0.0). Compared with Electron 34, Chromium moves from 132 to 134,
  Node.js from `20.18.1` to `22.14.0`, and V8 from 13.2 to 13.5.
- Electron 36 embeds Chromium `136.0.7103.48`, Node.js `22.14.0`, and V8
  `13.6` (36.0.0). Chromium moves from 134 to 136 and V8 from 13.5 to 13.6;
  Node.js stays at `22.14.0`.
- Electron 37 embeds Chromium `138.0.7204.35`, Node.js `22.16.0`, and V8
  `13.8` (37.0.0). Compared with Electron 36, Chromium moves from 136 to 138,
  Node.js from `22.14.0` to `22.16.0`, and V8 from 13.6 to 13.8.
- Electron 38 embeds Chromium `140.0.7339.41`, Node.js `22.18.0`, and V8
  `14.0` (38.0.0). Compared with Electron 37, Chromium moves from 138 to 140,
  Node.js from `22.16.0` to `22.18.0`, and V8 from 13.8 to 14.0.
- Electron 39 embeds Chromium `142.0.7444.52`, Node.js `22.20.0`, and V8
  `14.2` (39.0.0). Compared with Electron 38, Chromium moves from
  `140.0.7339.41` to `142.0.7444.52`, Node.js from `22.18.0` to `22.20.0`,
  and V8 from 14.0 to 14.2.
- Electron 40 embeds Chromium `144.0.7559.60`, Node.js `24.11.1`, and V8
  `14.4` (40.0.0). Compared with Electron 39, Chromium moves from
  `142.0.7444.52` to `144.0.7559.60`, Node.js from `22.20.0` to `24.11.1`,
  and V8 from 14.2 to 14.4.
- Electron 41 embeds Chromium `146.0.7680.65`, Node.js `24.14.0`, and V8
  `14.6` (41.0.0). Compared with Electron 40, Chromium moves from
  `144.0.7559.60` to `146.0.7680.65`, Node.js from `24.11.1` to `24.14.0`,
  and V8 from 14.4 to 14.6. The initial package was followed by high-priority
  fixes; choose 41.0.2 when upgrading to this line.
- Electron 42 embeds Chromium `148.0.7778.96`, Node.js `24.15.0`, and V8
  `14.8` (42.0.0). Compared with Electron 41, Chromium moves from
  `146.0.7680.65` to `148.0.7778.96`, Node.js from `24.14.0` to `24.15.0`,
  and V8 from 14.6 to 14.8.
- Electron 43 embeds Chromium `150.0.7871.46`, Node.js `24.17.0`, and V8
  `15.0` (43.0.0). Compared with Electron 42, Chromium moves from
  `148.0.7778.96` to `150.0.7871.46`, Node.js from `24.15.0` to `24.17.0`,
  and V8 from 14.8 to 15.0.

## Support lifecycle

- At the Electron 34 milestone, Electron 31 reached end of support; the
  supported lines were 34, 33, and 32.
- At the Electron 35 milestone, Electron 32 reached end of support; the
  supported lines were 35, 34, and 33.
- At the Electron 36 milestone, Electron 33 reached end of support; the
  supported lines were 36, 35, and 34.
- At the Electron 37 milestone, Electron 34 reached end of support; the
  supported lines were 37, 36, and 35.
- At the Electron 38 milestone, Electron 35 reached end of support; the
  supported lines were 38, 37, and 36.
- At the Electron 39 milestone, Electron 36 reached end of support; the
  supported lines were 39, 38, and 37.
- At the Electron 40 milestone, Electron 37 reached end of support; the
  supported lines were 40, 39, and 38.
- Electron 38 reached end of support with Electron 41.
- Electron 39 reached end of support with Electron 42.
- Electron 40 reached end of support with Electron 43.

## Operating-system and architecture floors

- Electron 38 and later require macOS 12 Monterey or later; macOS 11 Big Sur
  remains usable only with older Electron releases.
- Electron 44 requires macOS 13 Ventura or later and no longer supports macOS
  12 Monterey.
- Electron 43 is the last series that supplies prebuilt Electron binaries for
  Windows x86 (`win32-ia32`) and Linux ARMv7 (`linux-armv7l`). Support ends
  after that series reaches end of life in January 2027.
- Electron 44 also stops publishing 32-bit `chromedriver`, `mksnapshot`, and
  `ffmpeg` companion artifacts and the Windows x86 `node.lib` on the Electron
  headers CDN.

## Installing the Electron binary

Since Electron 42, the `electron` npm package downloads its binary on first
execution of its main bin script, not from `postinstall`. Installations may
therefore ignore scripts and fetch explicitly:

```sh
npm install electron --save-dev --ignore-scripts
npx install-electron
```

`ELECTRON_SKIP_BINARY_DOWNLOAD` is removed. Use
`ELECTRON_INSTALL_PLATFORM` and `ELECTRON_INSTALL_ARCH` to target a different
platform or architecture.

## Packaging-tool adjustments

- Electron Packager 19 enables ASAR packaging by default. This is separate
  from Electron 39's stable ASAR-integrity enforcement.
- Since Electron 40, macOS debug symbols are distributed as `dsym.tar.xz`
  rather than `dsym.zip`; update symbol ingestion and archive tooling.
