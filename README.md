# SD Card Sniffer

**English** · [Nederlands](README.nl.md)

Inspect SD cards and disk images, including the Linux filesystems that macOS
and Windows cannot mount themselves. Go backend (Wails v2) with a React
frontend.

The app opens a card read-only by default; it only writes when you explicitly
choose to.

**Contents:** [Download](#download) · [Install](#install) · [Usage](#usage) ·
[Cloning and writing](#cloning-and-writing) · [Assistant](#assistant) ·
[Languages](#languages-i18n) · [Building from source](#building-from-source) ·
[Making a release](#making-a-release) · [Development](#development) ·
[Layout](#layout)

## What it can read

| Filesystem | Browse | Notes |
|---|---|---|
| ext2 / ext3 / ext4 | yes | the root partition of a Raspberry Pi card |
| FAT12 / FAT16 / FAT32 | yes | the boot partition |
| ISO 9660 | yes | |
| SquashFS | yes | |
| btrfs, XFS, F2FS, NTFS, exFAT, APFS, HFS+ | no | detected and named |
| LUKS1 / LUKS2 | no | encrypted; unlock with `cryptsetup` |
| LVM2 | no | container; the volumes inside are not reachable this way |
| Linux swap | n/a | detected, contains no files |

Partition tables: GPT and MBR. A card without a partition table is treated as a
single volume.

## Download

Ready-made builds are on the
**[Releases page](https://github.com/baslentfert/bbsdcardsniffer/releases/latest)**:

| Platform | File | |
|---|---|---|
| macOS 10.13+ (Apple Silicon and Intel) | `bbsdcardsniffer-<version>-macos-universal.dmg` | disk image, drag to Applications |
| Windows 10/11 (64-bit) | `bbsdcardsniffer-<version>-windows-amd64-setup.exe` | installer with Start menu shortcut |
| Windows 10/11 (64-bit) | `bbsdcardsniffer-<version>-windows-amd64.exe` | standalone exe, no installation |

There is no ready-made Linux build (yet); see
[Building from source](#building-from-source).

## Install

### macOS

1. Open the `.dmg` and drag **bbsdcardsniffer** to **Applications**.
2. The app is not signed with an Apple Developer certificate, so the first time
   macOS refuses to open it ("Apple could not verify this app is free of
   malware"). Two ways to allow it once:
   - Try to open the app, then go to **System Settings → Privacy & Security**,
     scroll down and click **Open Anyway**; or
   - in Terminal:
     ```sh
     xattr -dr com.apple.quarantine /Applications/bbsdcardsniffer.app
     ```
3. Opening disk images now works with a double-click. Reading a **real SD
   card** requires root — see below.

Starting with root privileges (required for `/dev/rdiskN`):

```sh
sudo /Applications/bbsdcardsniffer.app/Contents/MacOS/bbsdcardsniffer
```

Without `sudo` the app still lists the cards, but refuses to open them with
*"no permission to read /dev/rdisk4"*.

### Windows

1. Run `…-setup.exe`. Because the installer is not code-signed, Windows
   SmartScreen shows *"Windows protected your PC"*: click **More info → Run
   anyway**.
2. Follow the installer. It adds a shortcut to the Start menu and the desktop.
3. The app asks for **Administrator** rights (UAC) every time it starts. That
   is required for raw access to `\\.\PhysicalDriveN`; without it, cards cannot
   be read.

The standalone `.exe` works the same way, just without installation and
shortcuts. WebView2 is embedded in the build, so nothing else needs to be
installed.

Uninstall via **Settings → Apps**.

> The write path (image → card) has not been tested on Windows yet; see
> [Cloning and writing](#cloning-and-writing). Reading and cloning to an image
> are non-destructive and safe to try.

### Claude API key (optional)

Only the [Assistant](#assistant) needs a key; browsing, cloning and writing
work without one. Create one at
[console.anthropic.com](https://console.anthropic.com/settings/keys) and enter
it in the app, or set it in your environment:

```sh
export ANTHROPIC_API_KEY=sk-ant-...
```

A key entered in the app is stored locally — never in the app itself or in
this repository:

| OS | Location |
|---|---|
| macOS | `~/Library/Application Support/bbsdcardsniffer/config.json` |
| Windows | `%AppData%\bbsdcardsniffer\config.json` |
| Linux | `~/.config/bbsdcardsniffer/config.json` |

The environment variable takes precedence over the file. Assistant usage is
billed to your own Anthropic account.

## Usage

Raw disk access requires elevated privileges (see [Install](#install)).
Optionally open a device or image right away by passing it as an argument:

```sh
# macOS
sudo /Applications/bbsdcardsniffer.app/Contents/MacOS/bbsdcardsniffer /dev/disk4
/Applications/bbsdcardsniffer.app/Contents/MacOS/bbsdcardsniffer ~/dumps/card.img

# Windows (from an Administrator prompt)
"C:\Program Files\Bas Lentfert\SD Card Sniffer\bbsdcardsniffer.exe" \\.\PhysicalDrive2

# Linux
sudo ./build/bin/bbsdcardsniffer /dev/sdb
```

An image file (for example a `dd` dump) does not need root and can also be
picked with the **Open disk image…** button.

On macOS a mounted card must be unmounted first. The **Unmount** button next to
the device does that via `diskutil unmountDisk`; the card stays in the reader.

## Cloning and writing

Besides browsing, the app has a **Copy** mode that works in two directions.

**Copy to an image.** Reads out to a file; the original is left untouched. The
source is whatever medium is open — a card or an opened image, so you can also
make a copy of a copy. Options:

- *Stop after the last partition* — skips the empty tail of a card whose rootfs
  has not been expanded yet. With GPT the app warns that the backup partition
  table then falls outside the image (`gdisk` repairs it).
- *Compress* to gzip.
- *Verify afterwards* — reads the image back and compares its SHA-256 with what
  came off the card.

**Image → card (writing).** The target is always the card selected on the left;
an opened image can only be the source. Overwrites the card completely.

Before writing, the target card is read and its contents are shown: partition
table, volumes, filesystem and label. If the card is not empty, the
confirmation names what will be lost — *"I understand that disk4 will be
erased, including boot, rootfs"* — instead of asking about an anonymous device.
A card that cannot be read is not presented as empty but as unreadable, with
the warning that it will be erased regardless.

If the image does not fit on the card, the button is blocked before anything
happens — as far as the size can be known up front. A raw image and a zip know
it; xz and gzip do not (gzip stores the length modulo 4 GiB, exactly in the
range where the answer matters), so there the check falls back to the moment of
writing. Supports raw images and `.gz`, `.xz`, `.bz2` and `.zip`; the format is
detected from the contents, not the extension, so a renamed file is never
written raw to the card.

Writing is deliberately locked down:

- `internal/imaging` refuses any target that is not listed as removable in the
  disk list, and also refuses a target that is not in the list at all. That is
  a hard refusal in the backend, not a dialog in the UI.
- The card is unmounted first (macOS `diskutil unmountDisk`, Linux `umount`,
  Windows `FSCTL_LOCK_VOLUME` + `FSCTL_DISMOUNT_VOLUME`).
- The UI asks for an explicit confirmation that names the target device; that
  confirmation is reset as soon as you pick a different card.
- An invalid target is refused before anything is torn down, so a mistake does
  not cost you the disk you had open.
- When done, the card is ejected on macOS, so the flush is guaranteed to be
  complete before it can be removed.

Both directions can be cancelled and show throughput and time remaining.

> The write path on **Windows** follows the documented lock/dismount sequence
> but has not been tested on Windows — no Windows machine was available. macOS
> and Linux follow the same pattern with `diskutil` and `umount` respectively.

## Assistant

A third mode puts a Claude agent on the opened volume. You ask a question in
plain language — "look in /var/log for why the network didn't come up" — and
the model calls the tools it needs by itself. That pattern is called *tool
use*; the loop around it an *agentic loop*.

The tool surface:

| Tool | Does | Modifies |
|---|---|---|
| `list_dir` | list directory contents | no |
| `read_file` | read a file, paginated; binary is reported, not dumped | no |
| `find` | search file names in the tree | no |
| `grep` | search lines in text files, with file and line number | no |
| `stat` | size, permissions, modification time | no |
| `write_file` | replace the full contents of a file | **yes** |
| `make_dir` | create a directory | **yes** |
| `delete` | delete a file or empty directory | **yes** |

The read tools are tailored to the diagnostic work this is mostly meant for:

- `read_file` takes a `tail` argument. Logs grow at the end, so the failure is
  at the back; paging forward through a 200 MB syslog is not a workable way to
  search.
- Rotated logs (`dmesg.4.gz`) are decompressed automatically. Without that they
  look binary and all history before the current file silently disappears.
- `grep` does not skip large files but searches their tail, and then marks the
  line numbers as approximate.
- The systemd journal (`/var/log/journal`) is a binary format this tool cannot
  decode. Instead of "binary file", the model is told *what* it is and where the
  text logs are, so its answer can pass that on.

By default the disk is read-only and the three write tools are **not even shown
to the model** — so it cannot reach for them. Only when you turn on *Allow
edits* is the medium reopened in write mode and do they become available. Every
call appears in the conversation, modifying calls highlighted in red, with the
full result expandable.

A key is required: `ANTHROPIC_API_KEY` in your environment, or entered once in
the app, after which it is stored in your user configuration with mode `0600`.

### Backup before editing

*Allow edits* does not switch over immediately, but first opens a gate that
offers to make a full copy of the medium — card or image, one button, verified
afterwards. Continuing without a copy is possible but requires an explicit
checkbox; it is your medium. Once a copy exists, the app remembers where it is
and names it in the warning bar for as long as writing is enabled.

That step is not there for show. The ext4 write path in go-diskfs is far less
proven than the read path, and we found four bugs in that read path. If
something goes wrong, you simply write the copy back — that path is tested.

## Languages (i18n)

The interface is available in English, Dutch and German; the language can be
picked in the top right and is remembered. **English is the default** — even
on a Dutch-language system. The terminology this app deals with (partition
table, superblock, filesystem) is English in every reference, and another
language should be a deliberate choice rather than a consequence of the machine
it happens to run on.

The catalogs live in [frontend/src/i18n/](frontend/src/i18n/):

| File | Role |
|---|---|
| `en.ts` | the English strings and the shape every language must match |
| `nl.ts`, `de.ts` | Dutch and German, both typed as `Messages` |
| `index.ts` | context, `useT()` hook, language choice and persistence |

Adding a language is one file next to `nl.ts` plus a line in `locales`.
Because every catalog is typed as `Messages`, a missing or misspelled key is a
**build error**, not a hole in the UI. The same goes for the list of example
questions: it is a fixed-length tuple, so adding a question in one language and
forgetting it in another fails `tsc`.

Strings with variables are functions rather than templates with placeholders,
so word order is up to the translator and not to English sentence structure.

The backend has its own catalog in [internal/i18n/](internal/i18n/), because
error messages are worded there. The frontend passes the language choice on
with `SetLocale`, so both halves speak the same language. The two catalogs do
not overlap: the frontend does the interface, Go the messages.

Go cannot flag a missing field as a build error the way TypeScript can, so that
gap is closed by a test that walks `Messages` with reflection and checks that
every language has every field, that no message is empty, and that no Dutch
string is identical to the English one — the latter is almost always a
forgotten translation.

Also **not** translated on the Go side: messages coming from the filesystem
drivers (`invalid checksum type 0`, `resource busy`). They are diagnostic
rather than actionable, they are easy to look up as-is, and rewriting the text
of an external library only makes that harder. They travel along as a detail
next to a translated explanation:

```
no permission to read /dev/rdisk4: run this application as root or
Administrator (open /dev/rdisk4: permission denied)
```

## Patched go-diskfs

`vendor/` contains a patched go-diskfs v1.9.4. Three checksum checks in the
ext4 reader assumed the `metadata_csum` feature is enabled, while every card
created with an older `mkfs.ext4` does not have that feature — and that is the
majority of cards in the field. Without the patches such a volume fails with
`invalid checksum type 0`.

The patch is in [patches/](patches/) and touches four files:

| File | Problem |
|---|---|
| `superblock.go` | `s_checksum_type` was rejected unconditionally if it is not 1, while the field is meaningless without `metadata_csum` and is never read again further on |
| `groupdescriptors.go` | the CRC16 variant for `gdt_csum` is implemented incorrectly (wrong CRC-16 variant, wrong seed, and the checksum is computed over its own field), so every descriptor mismatches |
| `inode.go` | inode checksums were always verified; every neighbouring check in that file is gated on `metadata_csum`, this one was not |
| `directoryentry.go` | `Info()` returned only the type bits, so every file showed up as `----------`; the permissions are in fact parsed from the inode |

Filesystems that do use `metadata_csum` are still fully verified, so real
corruption is still detected.

After changing dependencies, do not run `go mod vendor` but:

```sh
./scripts/vendor.sh
```

That re-vendors and applies the patches on top again.

## Development

```sh
wails dev      # live reload
wails build    # production build in build/bin
go test ./...  # backend tests
```

The tests build their own GPT image with a FAT32 boot and an ext4 root
partition, so no card is needed to run them. A sample image for the GUI:

```sh
SAMPLE_IMAGE=/tmp/card.img go test -run TestWriteSampleImage ./internal/volume/
```

Run a real card or image through it, with a report of what could be found and
read:

```sh
REAL_IMAGE=~/card.img go test -v -run TestRealImage ./internal/volume/
```

## Building from source

Requirements:

- [Go](https://go.dev/dl/) 1.25 or newer
- [Node.js](https://nodejs.org/) 20 or newer
- [Wails](https://wails.io/docs/gettingstarted/installation) v2.15:
  `go install github.com/wailsapp/wails/v2/cmd/wails@v2.15.0`
- Platform specific: on macOS the Xcode command line tools
  (`xcode-select --install`); on Linux `libgtk-3-dev` and
  `libwebkit2gtk-4.0-dev` (or `-4.1` with `-tags webkit2_41`); on Windows
  nothing extra, except [NSIS](https://nsis.sourceforge.io/) if you want to
  build the installer.

`wails doctor` checks that everything is in place.

```sh
git clone https://github.com/baslentfert/bbsdcardsniffer.git
cd bbsdcardsniffer
wails build                              # → build/bin/
wails build -platform darwin/universal   # macOS: Intel + Apple Silicon in one app
wails build -nsis -webview2 embed        # Windows: exe + installer
```

Dependencies are in `vendor/` (with patches, see
[Patched go-diskfs](#patched-go-diskfs)), so the Go build needs no network;
`npm install` for the frontend does.

## Making a release

Releases are built by GitHub Actions
([.github/workflows/release.yml](.github/workflows/release.yml)). Pushing a tag
is enough:

```sh
git tag v1.2.0
git push origin v1.2.0
```

The workflow builds the macOS `.dmg` (universal) on a macOS runner and the
Windows exe plus NSIS installer on a Windows runner, runs the tests, and
attaches the files to a new release with automatically generated release
notes. The version number from the tag ends up in `Info.plist` and the exe
metadata.

Starting it manually via **Actions → release → Run workflow** only builds the
artifacts (downloadable from the run), without a release.

The builds are not signed. Signing requires an Apple Developer ID (plus
notarization) and a Windows code-signing certificate; with those, the
Gatekeeper and SmartScreen warnings go away.

## Layout

| Path | Role |
|---|---|
| [app.go](app.go) | the methods bound to the frontend |
| [internal/blockdev/](internal/blockdev/) | raw device access with sector-aligned reads and a block cache |
| [internal/device/](internal/device/) | disk listing per platform (`diskutil`, `lsblk`, `Get-Disk`) |
| [internal/volume/](internal/volume/) | partition table, filesystem detection, browsing and exporting |
| [internal/imaging/](internal/imaging/) | cloning, writing, decompression and progress — the only thing that writes to a device |
| [internal/assistant/](internal/assistant/) | the Claude agent: tool surface and the agentic loop |
| [internal/config/](internal/config/) | API key storage |
| [assistant.go](assistant.go) | the assistant methods bound to the frontend |
| [transfer.go](transfer.go) | the clone and write methods bound to the frontend |
| [frontend/src/](frontend/src/) | React UI: device list, volume tabs, file browser, hex/text preview |

`internal/blockdev` opens read-only by default and its `Writable()` then
refuses; writing requires an explicit `OpenReadWrite`. Raw devices only accept
whole sectors, so a partial write becomes a read-modify-write of the
surrounding sectors, with invalidation of the block cache.

The aligned-read layer in `internal/blockdev` is not optional: `/dev/rdiskN` on
macOS and `\\.\PhysicalDriveN` on Windows reject reads that are not
sector-aligned, while the filesystem drivers read at arbitrary offsets.

## Third-party components

The licenses of all bundled Go and npm dependencies are in
[THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md); regenerate with
`./scripts/notices.sh`. The Nunito font is licensed under the
[SIL Open Font License](frontend/src/assets/fonts/OFL.txt).
