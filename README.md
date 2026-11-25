# reims-vm

Run macOS in a window on x86-64 Arch Linux, using a pinned build of [Reims](https://github.com/steelbrain/reims-vgpu). The arm64 guest is fully emulated, so expect it to run much slower than a real Mac.

Everything is built and stored inside this folder, in a Bubblewrap sandbox that sees the rest of the system read-only. Apart from the packages it needs, nothing is installed system-wide.

> **Legal.** Apple's license allows macOS only on Apple hardware. This repo ships scripts only; `create` downloads macOS from Apple. Whether you run it is your responsibility.

## Requirements

- An x86-64 PC running Arch Linux with a Wayland desktop.
- A GPU with a Vulkan driver (`nvidia-utils`, `vulkan-radeon` or `vulkan-intel`).
- 16 GB of RAM or more, and about 150 GB of free disk space.

Everything else is checked by `preflight` in step 2 below, which tells you how to fix anything missing.

<details>
<summary>What <code>preflight</code> checks</summary>

- **Packages.** It offers to install missing ones with `pacman`. `samba` is needed only if you share a folder. The full list:

  ```bash
  sudo pacman -S --needed bubblewrap git curl libarchive util-linux base-devel python ninja pkgconf flex bison bzip2 cmake clang glib2 pixman zlib pipewire libpulse alsa-lib libusb gnutls vulkan-icd-loader libx11 libxext libxrandr libxcursor libxi libxfixes libxss libxtst libxkbcommon wayland libdecor mesa libdrm dbus systemd-libs gtk3
  ```

- **`/dev/udmabuf`**, which the VM's GPU uses to share memory with the host. If it's missing: `sudo modprobe udmabuf`, add `udmabuf` to a file in `/etc/modules-load.d/`, and join the group that owns it (usually `kvm`).
- **`rustup`**: `sudo pacman -S rustup` if you have none. One from [rustup.rs](https://rustup.rs) or mise works too. The build downloads the Rust that Reims pins into `.state/` and leaves your toolchains and default alone.
- **The rest:** a Vulkan driver for your GPU, a GPU render node, Bubblewrap working, and enough free disk space.

</details>

## Getting started

1. **Get the code** and open a terminal in its folder:

   ```bash
   git clone https://github.com/KirillTregubov/reims-vm.git
   cd reims-vm
   ```

2. **Check your system:**

   ```bash
   bin/reims-vm preflight
   ```

   It offers to install any missing packages and says how to fix anything else. Run it again until it prints `preflight: ok`.

3. **Run the setup** from a terminal in your desktop session, so the VM can open its window:

   ```bash
   bin/reims-vm create
   ```

   Answer its questions, then leave it running until the VM window shows macOS setup.

4. **Set up macOS** in the VM window as on a new Mac. Skip the Apple ID sign-in; it fails on an emulated Mac.

From then on, `bin/reims-vm start` and `bin/reims-vm stop` are all you need. To run it from anywhere, link it onto your `PATH`: `ln -s "$PWD/bin/reims-vm" ~/.local/bin/`.

### What `create` does

| Stage           | How long                   | What happens                                                                                                                                                  |
| --------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Questions       | a minute                   | VM name, disk size, vCPUs, memory and an optional shared folder. The suggestions are half your CPU and memory. Skipped if you already have a `reims-vm.conf`. |
| Check and build | first time only, long      | Runs `preflight`, then builds Reims and QEMU.                                                                                                                 |
| Download        | depends on your connection | Downloads macOS from Apple, about 25 GB. Use `create 26B131` for a specific version, or `create /path/to/Restore.ipsw` for a file you have.                   |
| First boot      | about 5 minutes            | macOS creates its encrypted disk and restarts. The window then shows boot text and stops; that's expected. Leave it alone.                                    |
| Finish          | seconds                    | Completes the disk setup and boots into macOS setup.                                                                                                          |

If it's interrupted at any point, run `bin/reims-vm start`: it picks up where it left off.

## Commands

### Everyday

| Command                | What it does                                                                                                                                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `start`                | Boots the VM in a window, first completing an unfinished first boot. macOS follows the window's size as you resize or tile it. The pointer isn't captured, so you can leave the window freely. |
| `stop`                 | Shuts macOS down. See [Shutting down](#shutting-down).                                                                                                                                         |
| `list`                 | Shows the VM and whether it's running.                                                                                                                                                         |
| `share [DIR \| --off]` | Shows, sets or removes the shared folder. Applies from the next start.                                                                                                                         |
| `backup`               | Copies the stopped VM to `BACKUP_DIR` (default `backups/`).                                                                                                                                    |
| `config [edit]`        | Shows the settings in effect, or with `edit` asks for each setting, showing the current value.                                                                                                 |

### Setup and maintenance

| Command                  | What it does                                                                                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `create [VERSION\|IPSW]` | Sets everything up and boots into macOS setup (see above). Uses the latest macOS, a version such as `26B131`, or a local `.ipsw` file. Refuses if the VM exists.                                 |
| `preflight`              | Checks the host and offers to install missing packages.                                                                                                                                          |
| `install`                | Builds Reims and QEMU, which `create` does the first time. Rerun it after changing the `pin` in `bin/reims-vm` or `arch-packages.lock`. It skips finished steps and never disturbs a running VM. |
| `check`                  | Runs Reims's own host and GPU checks.                                                                                                                                                            |
| `diagnostics`            | Turns on the guest's boot log, for a black screen; restart the VM after. Turn it off again with `ctl set <VM> --logs false`.                                                                     |
| `ctl ARGS...`            | Runs any other `reims-ctl` command, such as `ctl show <VM>`.                                                                                                                                     |

## Shutting down

- **From macOS** (Apple menu → Shut Down): the safest. It waits for you to save open documents.
- **`bin/reims-vm stop`:** asks macOS to shut down, and powers the VM off if it hasn't after 90 seconds, losing unsaved work.
- **Closing the VM window:** powers the VM off immediately, like pulling the plug. Avoid it.

Shut the VM down before you reboot or log out of Linux.

## Settings

Settings live in `reims-vm.conf` in this folder. It's yours and isn't committed. Change it whichever way you prefer:

- **Edit the file.** Start from the documented example, uncomment what you want, and save:

  ```bash
  cp reims-vm.conf.example reims-vm.conf
  ```

  For example:

  ```bash
  VM=macos
  CPUS=6
  MEMORY=12G
  SHARE=/home/you/Shared
  ```

  It's a shell file: one `KEY=value` per line, full paths, and quotes around values with spaces.

- **Use the CLI.** `bin/reims-vm config edit` asks for each setting, showing the current value. `create` does the same the first time. These edit the file in place and keep your comments.

Every setting is optional: anything you leave out uses the default below. To see the values in use, run `bin/reims-vm config`.

| Key          | Default               | Meaning                                                                                                                                            |
| ------------ | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `VM`         | `macos`               | The VM's name. Set it before `create`; changing it later points at a different VM.                                                                 |
| `CPUS`       | half your CPU threads | vCPUs for the guest.                                                                                                                               |
| `MEMORY`     | half your RAM         | Guest memory, such as `12G`. Lower it if the host starts swapping.                                                                                 |
| `DISK`       | `128G`                | The VM's disk size. Set it before `create`; it can't be changed afterwards. The disk only takes the space macOS uses.                              |
| `SHARE`      | none                  | The folder shared with macOS. `share DIR` and `share --off` change just this.                                                                      |
| `SHARE_NAME` | the folder's name     | The name macOS connects to (`smb://10.0.2.4/<name>`). Saved when you first share a folder, so the address stays the same if you change the folder. |
| `BACKUP_DIR` | `backups/`            | Where `backup` writes. Must be on the same filesystem as this folder, or `backup` fails.                                                           |

Changes take effect the next time the VM starts. If you mistype a value in the file, such as `CPUS=99` on a 12-thread machine, `start` stops and tells you which setting is wrong and what it accepts.

## Sharing a folder

```bash
sudo pacman -S --needed samba
bin/reims-vm share ~/path/to/folder
```

Start (or restart) the VM, then in macOS open Finder → Go → Connect to Server, enter `smb://10.0.2.4/<SHARE_NAME>` (`start` prints it; it's the folder's name unless you change it) and connect as Guest. Files macOS writes are owned by you. To reconnect at every login, drag the mounted share into System Settings → General → Login Items.

QEMU serves the folder to the VM only, through a private Samba server that runs as you while the VM runs; nothing else on your computer or network can reach it. Its log is `.state/smbd.log`.

## Backups

With the VM stopped, run `bin/reims-vm backup`. It needs btrfs or XFS, where the copy is instant and takes no extra space until macOS changes the disk.

To restore a backup, stop the VM and copy the backup's files back into `.state/data/reims-ctl/vms/<VM>/` with `cp -a --reflink=always`.

A backup is on the same drive as the VM, so it protects against mistakes, not against a failed drive. To keep a copy elsewhere, use `cp --sparse=always` or `rsync -S`, so the 128 GB sparse disk stays small.

## Where things are stored

Everything stays in this folder and is git-ignored. Never delete `.state/` as a whole, and avoid `git clean -x`: both delete the VM.

<details>
<summary>What's where</summary>

| Path                                                     | Contents                                                                                            |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `.state/data/reims-ctl/vms/<VM>/`                        | **The VM**: macOS disk, NVRAM and settings. The only part that can't be rebuilt.                    |
| `.state/data/reims-ctl/installers/`                      | The downloaded macOS restore image, about 25 GB. Delete it if you won't `create` again.             |
| `.state/data/reims-ctl/qemu/`                            | QEMU builds. `current` is the newest.                                                               |
| `.state/install/bin/`                                    | `reims-ctl` and its helpers.                                                                        |
| `.state/config/vms.toml`                                 | Reims's list of VMs. Keep it with the VM.                                                           |
| `.state/{reims-vgpu,cache,rustup,cargo,arch,downloads}/` | Source, build caches, Rust toolchain and pinned packages. Safe to delete; `install` recreates them. |
| `backups/`                                               | Backups.                                                                                            |
| `reims-vm.conf`                                          | Your settings; `reims-vm.conf.example` documents every key.                                         |

</details>

## Troubleshooting

- **Text over the screen while booting:** that's the firmware's boot log. It's normal and goes away once macOS starts.
- **The VM feels frozen:** it's usually busy, or the host is swapping. Check host memory with `free -h`, close heavy apps, or lower `MEMORY` with `config edit`. If it stays stuck, use `stop`, not the window's close button.
- **Black screen:** run `diagnostics`, restart the VM, and read the logs in `.state/data/reims-ctl/vms/<VM>/logs/`.
- **Shared folder won't connect:** check that `samba` is installed, and read `.state/smbd.log`.
- **Several GPUs:** Reims uses a discrete GPU before an integrated one. To pick another, list them with `ctl gpus`, then run `ctl set <VM> --gpu <number>` and restart the VM.

## Uninstall

Stop the VM and delete this folder. Nothing else was installed except the packages `preflight` asked for.
