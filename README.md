# iOS 26 on the Xiaomi 15 — build and run instructions

Run Apple's iOS 26 (arm64e) on a Xiaomi 15 (codename **dada**, Snapdragon 8 Elite / SM8750),
using the open-source project **[superboyyy/hakios-arm64e-dada](https://github.com/superboyyy/hakios-arm64e-dada)**.

The project boots Apple's open-source XNU kernel (xnu-12377.121.6, the iOS 26 kernel) on the phone
and runs the unmodified iOS 26 arm64e user space on top of it. It is built on HakiOS and UniXNU work.

> **Warning — read first.**
> This is experimental research code, not a ROM. It requires an **unlocked bootloader** (which wipes
> the phone and, for HyperOS, involves Xiaomi's unlock process and waiting period). Mistakes can cost
> you your data or brick the phone. Only try it if you are comfortable tinkering and can accept the
> risk. Nothing here is affiliated with Apple, Qualcomm or Xiaomi.

## What you get (project status as of Nov 2026)

| Part | Status |
| --- | --- |
| CPU | All 8 Oryon cores natively, hardware PAC, up to 3.53/4.09 GHz over SCMI, no instruction translation |
| Boot | Custom EFI loader started from Mu-Silicium UEFI; iOS image lives in an Android DSU partition |
| System | iOS 26 reaches the home screen; touch works; flash storage read/written on demand |
| GPU | Adreno 830 via Metal → metal2vulkan → Vulkan → Turnip → custom kernel driver |
| Liquid Glass | Core shaders translate correctly (pixel-compared against Apple GPU); some issues remain |
| Audio | Speaker playback works (ADSP, AudioReach, CS35L43 amps); microphones in progress |
| Battery | Fuel gauge reported to iOS |
| Bluetooth | Works (WCN7850-family over UART, given to iOS's bluetoothd) |
| Wi-Fi | Gets as far as WMI; firmware dies in most runs (driver is a temporary stand-in) |
| HDR, camera, modem | Not started |

**Not a daily driver:** no cellular, no charging under the custom kernel, no camera, no App Store
reliability guarantees, no Secure Enclave.

## Requirements

### Hardware
- **Xiaomi 15 (dada)** — the global/China Snapdragon 8 Elite (SM8750) model, 12 GB or 16 GB RAM.
  The 12/16 GB variants are what the project targets; no other phone will work with these exact files.
- A PC with USB, plus `adb` and `fastboot` (Android platform tools).

### Phone preparation
- Bootloader unlocked (HyperOS: enable *OEM unlocking* and *USB debugging* in Developer options,
  bind the device to your Mi account, use Xiaomi's unlock tool; this **wipes the phone**).
- The iOS system image is stored on the phone as an **Android DSU (Dynamic System Updates)** image,
  so Android remains the daily OS and the phone still boots Android normally. The DSU install
  script sets `persist.sys.fflag.override.settings_dynamic_system` over adb — on some stock HyperOS
  builds adb cannot set a `persist.sys.*` property without root. **Gap:** if the DSU install fails at
  that step, temporary root (e.g. Magisk on the boot image) is the likely fix.

### Build machine
- Linux x86-64 (the scripts are written for it; the kernel build uses clang/lld and the HakiOS
  Linux XNU toolchain). **Note:** decrypting the iOS 26 `.aea` filesystem archives is documented in
  the project with macOS's `/usr/bin/aea` plus Apple's FCS key, so access to a Mac (at least for the
  IPSW extraction step) is currently expected.
- ~60+ GB free disk (IPSW, extracted caches, build trees, 24 GiB DSU slot on the phone).
- Packages: `git`, `python3`, `curl`, `7z` (7-Zip **23 or newer** — older versions cannot read APFS),
  `gcc`, `clang`, `lld`/`lld-link`, `docker`.
- For the kernel build: the HakiOS `xnu-deps` toolchain (theos-style cctools + an iPhoneOS SDK).
  See `base/xnu/README.md` in the source repo for exact setup.

## Step 1 — Get the source

```sh
git clone https://github.com/superboyyy/hakios-arm64e-dada
cd hakios-arm64e-dada
```

Layout: `kernel/` (XNU patch), `base/` (HakiOS kernel tree: loader, build/flash tools, device configs),
`userland/` (iOS 26 bring-up, injection, Metal plug-in, audio HAL), `uefi-dada/` (UEFI patch + Docker
build), `gpu2/` (metal2vulkan / Turnip patches). The repository contains **no Apple, Qualcomm or
Xiaomi binaries** — you fetch those yourself in the next steps.

## Step 2 — Fetch and decrypt iOS 26

The project targets **iOS 26.6.2 (build 23G90)** for a recent arm64e iPhone (iPhone 17 Pro family;
the scripts reference the `23G90__iPhone18,1` extracted-cache directory, and the porting notes
reference iPhone18,2 — either way it is the iPhone 17 Pro-line IPSW, arm64e).

iOS 26 ships its root and cryptex filesystems as `.aea` (Apple Encrypted Archive). Per the project's
porting notes (`base/docs/ARM64E.md`):

1. Download the iPhone 17 Pro-line iOS 26.6.2 (23G90) IPSW from Apple.
2. Decrypt the `.aea` archives with macOS `/usr/bin/aea` and the FCS key from Apple.
3. Extract the complete **arm64e dyld shared cache** (84 subcaches + `.atlas` + `.symbols`) from the
   `Cryptex1,SystemOS` cryptex, plus the root filesystem.
4. Put the result in a directory **next to** the repo, named `ios26pro`, so that
   `../ios26pro/23G90__iPhone18,1/dyld_shared_cache_arm64e*` exists relative to `userland/`.

**Gap:** the repo does not ship a ready-made fetch script for iOS 26 the way upstream HakiOS ships
`ios165/fetch.sh` for iOS 16.5. The extraction above is reconstructed from the porting notes; exact
`aea` flags and key retrieval are on you.

## Step 3 — Pull the phone's own firmware

Vendor firmware cannot be redistributed, so it is taken from the phone itself over adb (phone booted
in Android, USB debugging on). For dada, the audio path needs the ADSP firmware in `fw-adsp-dada/`
(`ADSP=1` in the image build puts it into the small image). Upstream HakiOS pulls analogous firmware
(GPU/Wi-Fi/ADSP/haptics) from `/vendor/firmware*` — check what your HyperOS build exposes and mirror
the layout the build scripts ask for (`userland/build.sh` prints which firmware went in).

**Gap:** the exact dada firmware file list is not enumerated in the public repo; the upstream
redwood instructions and the build scripts' complaints are the guide.

## Step 4 — Build the kernel

```sh
# make the dada kernel tree: Apple's xnu-12377.121.6 + the dada patch
kernel/apply.sh ~/hakios-gpu2/xnu12377

# build it (expects the xnu-deps toolchain; XNU12377 env overrides the source path)
base/tools/dada-build.sh kernel
```

Produces `~/hakios-gpu2/xnu12377/BUILD/obj/mach.development.bcm2837` (arm64e, standard-PAC backend).

## Step 5 — Build the UEFI + loader boot image

Nothing is flashed to the phone — the whole chain boots from RAM via `fastboot boot`.

```sh
# fetch the dada UEFI base (7Night-kqx/uefi-dada at the pinned commit) and apply the patch
uefi-dada/apply.sh                                    # -> $HOME/dada-uefi-build

# build the UEFI in Docker
docker build -t hakios-uefi -f uefi-dada/Dockerfile $HOME/dada-uefi-build
docker run --rm -v $HOME/dada-uefi-build:/work -w /work hakios-uefi \
    python3 build_uefi.py -d dada -r RELEASE          # -> Build/dadaPkg/RELEASE_CLANGPDB/FV/SILICIUM_UEFI.fd

# build the XNU loader and pack shim + UEFI + loader into out/dada-usb.img
base/tools/dada-build.sh loader
```

## Step 6 — Build the iOS images

```sh
cd userland
ADSP=1 REST=1 OSLOG="" IOS=../ios26pro ARCH=arm64e \
    CACHE=../ios26pro/23G90__iPhone18,1 RC=rc.launchd26.ca ./build.sh
cd ..
```

This produces `userland/root.apfs` (small boot image with the custom init and launchd jobs) and
`userland/system.apfs` (the big iOS system image, ~5 GB).

## Step 7 — Put the system image on the phone (DSU)

Phone booted in Android, adb connected:

```sh
base/tools/dada-dsu.sh install userland/system.apfs
```

This pushes the image through Android's own DSU installer into a 24 GiB slot. It is **never
enabled**, so the phone keeps booting Android; the loader just reads it from storage (~1.5 s instead
of a ~130 s USB upload). Undo at any time: *Settings → Developer options → DSU → Discard*.

Later image updates don't need a full reinstall:

```sh
base/tools/dada-dsu.sh patch userland/system.apfs   # sends only changed 1 MB pieces
```

Optional but recommended before experiments — back up whole GPT partitions through the loader
(read-only, phone in xnuloader):

```sh
base/tools/dada-backup-part.sh backup NAME:SIZEHEX...
```

## Step 8 — Run iOS 26

```sh
base/tools/dada-ios.sh ios 120
```

One line does everything: `dada-cycle.sh` brings the phone from Android → bootloader → xnuloader via
`fastboot boot out/dada-usb.img`, the loader reads the DSU image from storage, the kernel boots with
the full working configuration (1206×2622 display, touch, buttons, Adreno 830 GPU with Metal
compositing, ADSP + speakers, battery gauge, all 8 cores, Bluetooth, mic), and the phone **reboots
itself back to Android** when the session ends (here after 120 s; raise the number for longer).

Useful variations:

```sh
CPUS=8 base/tools/dada-ios.sh ios 600                       # longer session
base/tools/dada-ios.sh ios 120 dadawifi=4                   # try Wi-Fi (unstable; see below)
base/tools/dada-screenshot.sh shot.png                      # grab the framebuffer after a run
KEEP_POLLS=1 base/tools/dada-ios.sh ios 120                 # keep every live log poll
```

Logs land in `logs/<name>-<time>.log` next to the repo's `base/` tree, with `.full` / `.stop.log`
variants on panics or early stops. A kernel that crashes the phone is recovered by the watchdog —
the phone returns to Android on its own.

## Known limitations and troubleshooting

- **Wi-Fi:** `dadawifi=4` (passed **first** on the command line) brings the ath12k-based driver up
  through MHI/QMI/calibration and has scanned real networks, but the firmware dies at its first
  radio work in most runs. The current Wi-Fi driver is a temporary stand-in.
- **No modem, no camera, no HDR.** No cellular of any kind.
- **Microphones:** in progress (`dadamic=3` is already in the default line; your mileage may vary).
- **Charging:** the phone discharges while running the custom kernel. Watch the battery.
- **The phone doesn't come back to Android:** the loader watchdog resets it automatically; if it
  seems stuck, wait ~5 minutes before power-cycling by hand.
- **DSU install never starts:** check `base/tools/dada-dsu.sh status`, and see the root note in
  "Phone preparation" above.
- Debugging hooks built into `dada-run.sh`: `STOP_ON=<regex>` (end the run when the log matches),
  `KEEP=1` (leave the loader up), `CRC=1` (verify the uploaded image), `rec_secs=N` boot-arg
  (screen recording to `.mp4`).

## Safety notes

- Nothing is flashed: the boot chain lives in RAM (`fastboot boot`), and the only on-disk change is
  the DSU image inside Android's own DSU slot, which Android can discard.
- iOS writes can persist (`var_store` / `ufs_write` work on this stack) — treat the phone's storage
  accordingly.
- Keep a backup before you start (`dada-backup-part.sh`), and know how to reflash stock HyperOS
  fastboot images for your exact model before experimenting.

## Sources and credits

- Project: <https://github.com/superboyyy/hakios-arm64e-dada> (status table, scripts and quotes above
  are taken from its README and `base/tools/dada-*.sh`, `base/common/devices/dada.conf`,
  `base/docs/ARM64E.md`, `uefi-dada/README.md`)
- Built on: **HakiOS** (OOSEX GANG), **UniXNU** (NativeAdreno/NativeMetal, Ath12kWifi),
  **Data A Live**, **metal2vulkan**, **Mesa (Turnip/freedreno)**, **Project/Mu-Silicium**,
  **7Night-kqx/uefi-dada**.
- Licences: kernel patch APSL 2.0; UEFI patch BSD-2-Clause-Patent; GPU patches keep their upstream
  licences (see `NOTICE.md` in the source repo).

*Instructions compiled 2026-11-19. Marked "Gap" where the public repo does not fully document a step.*
