# AGENTS.md: Lily58 wireless (nice!view) ZMK config

This repo is Mark's ZMK config for a wireless Lily58. Each half has a **nice!nano v2** controller and a **nice!view** display. GitHub Actions builds the firmware, so nothing is compiled locally.

- **This repo:** `mark-vella/lily58-wireless-view-zmk-config` (`origin`, branch `main`).
- **Upstream template:** `typeractivexyz/lily58-wireless-view-zmk-config`. This repo was created from the template and is **not** a GitHub fork, so there is no `upstream` remote. Pull upstream changes by URL (see below).
- **Keymap Editor:** https://nickcoutsos.github.io/keymap-editor/ is a browser GUI that uses the GitHub integration to commit straight to `main` as `keymap-editor[bot]`.

## Layout

| File | Purpose |
| --- | --- |
| `config/lily58.keymap` | Keymap (devicetree): layers, combos, behaviour tweaks (`&mt` flavor/tapping-term) |
| `config/lily58.conf` | Kconfig: sleep timeouts, BT TX power, debounce |
| `config/west.yml` | Pins the ZMK version (`revision: v0.3`) |
| `.github/workflows/build.yml` | Calls ZMK's reusable build workflow (`build-user-config.yml@v0.3`) |
| `build.yaml` | Build matrix: left and right halves. Only the left half gets the `studio-rpc-usb-uart` snippet (ZMK Studio over USB) |

The **left half is the central half**. It holds the keymap and talks to the host. The right half is a peripheral that only sends key positions to the left.

## Editing the config

Changes can come from three places. Always sync before editing, because the Keymap Editor may have pushed commits:

```sh
git pull --ff-only origin main
```

### 1. Editing the files by hand (agents should use this)
- The Lily58 has **58 positions per layer**: 4 rows of 6+6, a bottom row of 6+6 plus 2 inner keys (`[`/`]` on layer 0), and 4+4 thumb keys. Every `bindings = < ... >;` block needs exactly 58 entries. Keep the column alignment.
- Keep each layer's ASCII-art comment in sync with its bindings, and keep `display-name` set. Studio and the nice!view show the name.
- Use only keycodes and behaviours from ZMK **v0.3** (https://zmk.dev/docs/keymaps/list-of-keycodes, https://zmk.dev/docs/keymaps/behaviors). Newer docs may describe features that v0.3 doesn't have.
- Keep `&studio_unlock` reachable. It's currently on layer 2, home row, right side.
- Keep a way to switch output (`&out OUT_USB` / `OUT_BLE`) and Bluetooth profile (`&bt BT_SEL n`) somewhere in the keymap.
- There's no local build. The check is CI: push a branch or open a PR, then run `gh run watch` / `gh run list`. A red build usually means a wrong binding count or an unknown keycode, and `gh run view --log-failed` shows the devicetree error.
- Use small commits with a descriptive message. PRs into `main` are the usual flow.

### 2. Keymap Editor (the user's GUI)
- It loads `config/lily58.keymap` from GitHub and commits back to `main` directly, so pull afterwards.
- It may reformat the file. Check the diff for lost comments or alignment when editing by hand after it.

### 3. ZMK Studio (live, no flashing)
- Studio is enabled on the left half. Connect it over USB, open https://zmk.studio, then press `&studio_unlock`.
- **Changes saved in Studio are stored on the keyboard and override the keymap in this repo, even after you reflash.** If a flashed keymap change "doesn't take", that's the likely cause. Use Studio's **Restore Stock Settings** to go back to the keymap that was compiled in.
- Studio changes are never written back to the repo. To make one permanent, copy it into `lily58.keymap` by hand.

## Flashing

### 1. Get the firmware from CI
Use the latest successful `main` build:

```sh
gh run list -L 5                 # find the run id, status must be success
d=$(mktemp -d /tmp/lily58-fw.XXXX)
gh run download <run-id> -D "$d" # gives $d/firmware/*.uf2
```

Files:
- `lily58_left nice_view_adapter nice_view-nice_nano_v2-zmk.uf2`
- `lily58_right nice_view_adapter nice_view-nice_nano_v2-zmk.uf2`

### 2. Flash one half at a time
The user plugs in a half over USB and double-taps its reset button. The nice!nano bootloader then mounts as a USB drive labelled `NICENANO`, at `/run/media/$USER/NICENANO` on this machine. Before copying, check that the drive is there and that it's the bootloader:

```sh
findmnt -n /run/media/$USER/NICENANO && cat /run/media/$USER/NICENANO/INFO_UF2.TXT   # Board-ID: nRF52840-nicenano-v2
cp "$d/firmware/lily58_left nice_view_adapter nice_view-nice_nano_v2-zmk.uf2" /run/media/$USER/NICENANO/
sync
```

- **Copying is the flash.** The board writes the firmware, reboots and unmounts itself. A "drive removed unexpectedly" warning or a `cp`/`sync` error from the drive disappearing is normal. Check success by confirming `NICENANO` is gone (`lsblk -o NAME,LABEL`).
- Then the user plugs in the other half and double-taps reset. Copy the **right** file onto it the same way.
- **Never put the left file on the right half, or the right file on the left.** If you're not sure which half is plugged in, ask.
- Only one `NICENANO` should be mounted at a time.
- **The order doesn't matter.** Each half is flashed on its own, so left-then-right is a habit rather than a rule. Flash whichever half is plugged in, using the file for that half. The order of turning them on afterwards doesn't matter either: the right half advertises until the left half connects.

### Which halves need flashing
- **Keymap-only changes:** strictly only the **left** half needs flashing. Flashing both is harmless and keeps them on the same build.
- **`.conf` changes, ZMK version bumps, or anything affecting both halves:** flash **both**.

### If the halves won't pair after flashing
Do a settings reset:
1. Add `settings_reset` to `build.yaml` temporarily:
   ```yaml
     - board: nice_nano_v2
       shield: settings_reset
   ```
2. Flash `settings_reset-nice_nano_v2-zmk.uf2` to **both** halves, then flash the normal firmware to both again.
3. Press reset on both halves at about the same time so they pair again.
4. Remove the host's Bluetooth pairing for the keyboard and pair it again.

## Keeping up to date

### ZMK firmware version
ZMK is pinned in **two places, which must always match**:
- `config/west.yml`: `revision: v0.3`
- `.github/workflows/build.yml`: `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`

To upgrade:
1. Check for a newer release with `git ls-remote --tags https://github.com/zmkfirmware/zmk` or at https://github.com/zmkfirmware/zmk/releases.
2. Read the release notes and the blog (https://zmk.dev/blog) for breaking changes: renamed keycodes or behaviours, Kconfig options, shield names, snippet names.
3. On a branch, bump both pins, fix any keymap or conf breakage, push, and wait for a green CI run.
4. Flash **both halves**. The split halves must run the same ZMK version. If they won't pair, do the settings reset above.

Don't track ZMK `main`. Stay on tagged releases.

### Upstream template (typeractivexyz)
Check for new upstream commits now and then. The template mostly changes `build.yaml`, `build.yml`, `west.yml` and the `.conf` defaults:

```sh
git fetch https://github.com/typeractivexyz/lily58-wireless-view-zmk-config main
git log --oneline HEAD..FETCH_HEAD   # anything new?
git diff HEAD...FETCH_HEAD           # review it
git merge FETCH_HEAD                 # on a branch
```

- If there are conflicts in `config/lily58.keymap`, **keep Mark's keymap**. The upstream keymap is only a stock example.
- Take upstream changes to the build and infrastructure files (ZMK pin, workflow, snippets, shields) unless they conflict with something Mark changed on purpose.

## Don'ts
- Don't flash anything without confirming that a `NICENANO` bootloader drive is mounted and which half it is.
- Don't force-push `main`, because the Keymap Editor commits there.
- Don't bump the ZMK version in only one of the two places.
- Don't remove the `studio-rpc-usb-uart` snippet or `&studio_unlock` unless the user asks.
