# Build Instructions

This guide covers how to set up the development environment and build Handy from source across different platforms.

## Prerequisites

### All Platforms

- [Rust](https://rustup.rs/) (latest stable)
- [Bun](https://bun.sh/) package manager
- [Tauri Prerequisites](https://tauri.app/start/prerequisites/)

### Platform-Specific Requirements

#### macOS

- Xcode Command Line Tools
- Install with: `xcode-select --install`

##### Intel Mac (x86_64)

Prebuilt ONNX Runtime binaries are not available for Intel Macs. Install ONNX Runtime via Homebrew and link dynamically:

```bash
brew install onnxruntime
ORT_LIB_LOCATION=$(brew --prefix onnxruntime)/lib ORT_PREFER_DYNAMIC_LINK=1 bun run tauri dev
```

The same environment variables apply for production builds:

```bash
ORT_LIB_LOCATION=$(brew --prefix onnxruntime)/lib ORT_PREFER_DYNAMIC_LINK=1 bun run tauri build
```

#### Windows

- Microsoft C++ Build Tools
- Visual Studio 2019/2022 with C++ development tools
- Or Visual Studio Build Tools 2019/2022

#### Linux

- Build essentials
- ALSA development libraries
- Install with:

  ```bash
  # Ubuntu/Debian
  sudo apt update
  sudo apt install build-essential libasound2-dev pkg-config libssl-dev libvulkan-dev vulkan-tools glslc libgtk-3-dev libwebkit2gtk-4.1-dev libayatana-appindicator3-dev librsvg2-dev libgtk-layer-shell0 libgtk-layer-shell-dev patchelf cmake

  # Fedora/RHEL
  sudo dnf groupinstall "Development Tools"
  sudo dnf install alsa-lib-devel pkgconf openssl-devel vulkan-devel \
    gtk3-devel webkit2gtk4.1-devel libappindicator-gtk3-devel librsvg2-devel \
    gtk-layer-shell gtk-layer-shell-devel \
    cmake

  # Arch Linux
  sudo pacman -S base-devel alsa-lib pkgconf openssl vulkan-devel \
    gtk3 webkit2gtk-4.1 libappindicator-gtk3 librsvg gtk-layer-shell \
    cmake clang xdotool bun pipewire-pulse pipewire-alsa pipewire-audio wireplumber
  ```

> **Note on Audio Subsystem (PipeWire):** Ensure the user audio session services are enabled:
> ```bash
> systemctl --user enable --now pipewire pipewire-pulse wireplumber
> ```

> **Note for bindgen / Whisper build:** Make sure `libclang.so` is discoverable during compilation. On Linux/Arch:
> ```bash
> export LIBCLANG_PATH=/usr/lib
> ```

## Setup Instructions

### 1. Clone the Repository

```bash
git clone git@github.com:gabrieldasf/qtz-handy.git
cd qtz-handy
```

### 2. Install Dependencies

```bash
bun install
```

### 3. Start Dev Server

```bash
bun tauri dev
```

### 4. Build for Production

> ⚠️ **Important:** Do NOT use raw `cargo build --release` directly. Raw cargo compiles with `devUrl: "http://localhost:1420"`, causing `Could not connect to localhost: Connection refused` in production. Always build using the Tauri CLI so frontend assets in `dist/` are embedded into the binary:

```bash
export LIBCLANG_PATH=/usr/lib

# Option 1: Generate deb bundle + release binary
./node_modules/.bin/tauri build -b deb --ci

# Option 2: Generate standalone release binary without package bundle
./node_modules/.bin/tauri build --no-bundle --ci
```

The compiled binary will be at `src-tauri/target/release/handy`.

## Linux Install (from source)

Handy supports both user-local installation (no root/sudo required) and system-wide installation:

### Option A: User-Local Install (Recommended / No Sudo)

Install directly to `~/Applications/Handy` and link into `~/.local/bin`:

```bash
# 1. Create target directories
mkdir -p ~/Applications/Handy ~/.local/bin ~/.local/share/applications ~/.local/share/icons/hicolor

# 2. Extract files from the deb bundle
cd /tmp
ar x /path/to/qtz-handy/src-tauri/target/release/bundle/deb/Handy_*_amd64.deb data.tar.gz
tar xzf data.tar.gz

# 3. Copy binary and bundled resources alongside executable
cp usr/bin/handy ~/Applications/Handy/
cp -r usr/lib/Handy/resources ~/Applications/Handy/
cp -r usr/share/icons/hicolor/* ~/.local/share/icons/hicolor/

# 4. Create PATH symlink
ln -sf ~/Applications/Handy/handy ~/.local/bin/handy

# 5. Create desktop menu entry
cat << 'EOF' > ~/.local/share/applications/Handy.desktop
[Desktop Entry]
Categories=Utility;AudioVideo;
Comment=Speech-to-text desktop utility
Exec=/home/$USER/Applications/Handy/handy
StartupWMClass=handy
Icon=handy
Name=Handy
Terminal=false
Type=Application
EOF

update-desktop-database ~/.local/share/applications 2>/dev/null || true
gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor 2>/dev/null || true
```

### Option B: System-Wide Install (Requires Sudo)

```bash
cd /tmp
ar x /path/to/qtz-handy/src-tauri/target/release/bundle/deb/Handy_*_amd64.deb data.tar.gz
tar xzf data.tar.gz
sudo cp usr/bin/handy /usr/bin/
sudo cp -r usr/lib/Handy /usr/lib/
sudo cp -r usr/share/icons/hicolor/* /usr/share/icons/hicolor/
sudo cp usr/share/applications/Handy.desktop /usr/share/applications/
sudo update-desktop-database 2>/dev/null || true
```

After subsequent rebuilds, only the binary needs re-copying to `~/Applications/Handy/handy` or `/usr/bin/handy`.

Resources only need re-copying if they change upstream (new icons, sounds, etc.).

## Troubleshooting

### AppImage build fails on Arch / rolling-release distros

`linuxdeploy` bundles its own `strip` binary which is too old to process system libraries built with newer toolchains on rolling-release distros (Arch, CachyOS, Manjaro, EndeavourOS).

The error from Tauri:

```
Bundling Handy_*_amd64.AppImage
failed to bundle project `failed to run linuxdeploy`
```

Tauri swallows the real linuxdeploy error. To see it, run linuxdeploy manually:

```bash
cd src-tauri/target/release/bundle/appimage
~/.cache/tauri/linuxdeploy-x86_64.AppImage --appimage-extract-and-run \
  --appdir Handy.AppDir --plugin gtk --output appimage
```

**Workaround:** The binary, deb, and rpm bundles all build fine — only the AppImage step fails. To skip it:

```bash
bun run tauri build -- --bundles deb
```

Then install using the deb extraction method above.

## Windows (QTZ / this machine)

This checkout (qtz-handy) has extra native dependencies (whisper via bindgen + ort + cpal).

### One-command build + open installer (recommended)

1. Open **PowerShell as Administrator**.
2. Run:

```powershell
cd D:\Apps\QTZ-Apps\qtz-handy
.\build-installer.ps1
```

The script will:
- Install LLVM (for libclang) via winget/choco if missing
- Configure MSVC environment (vcvars64)
- Run the full `bun run tauri build`
- Automatically locate the .msi (or .exe) and open it

### Manual steps (if the script needs tweaks)

```powershell
# As Administrator
choco install llvm -y

$env:LIBCLANG_PATH = "C:\ProgramData\chocolatey\lib\llvm\tools\LLVM\bin"
cd D:\Apps\QTZ-Apps\qtz-handy

# Ensure MSVC vars + build
cmd /c "call `"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat`" >nul && bun run tauri build"

# Open the installer
$msi = Get-ChildItem "src-tauri\target\release\bundle\msi\Handy_*.msi" | Select -First 1
Invoke-Item $msi.FullName
```

The resulting installer is normally at:
`src-tauri/target/release/bundle/msi/Handy_0.8.3_x64_en-US.msi`

(Also generates an NSIS .exe in the same tree.)

