# Installing Whisky Wine 3.0

Whisky Wine 3.0 is built on Wine 11.0 with CrossOver 26.0.0 patches, GPTK 3.0,
and DXVK. It is built via GitHub Actions and distributed as a `Libraries.tar.gz`
artifact.

## Requirements

- Apple Silicon Mac running macOS 15.0 (Sequoia) or later
- 16 GB RAM or more recommended
- [Whisky.app](https://github.com/Whisky-App/Whisky)

## Installing from CI artifacts

### 1. Download the artifact

From the GitHub Actions UI:

1. Go to the **Actions** tab of this repository.
2. Select a successful **Build Wine** workflow run.
3. Under **Artifacts**, download `Libraries`.

The download contains `Libraries.tar.gz`.

Alternatively, use the GitHub CLI:

```sh
# List recent workflow runs
gh run list --workflow=build.yml

# Download the artifact from a specific run
gh run download <run-id> -n Libraries
```

### 2. Extract to the Whisky Libraries directory

```sh
# Remove any existing Libraries directory
rm -rf ~/Library/Application\ Support/com.isaacmarovitz.Whisky/Libraries

# Extract the artifact
tar -xzf Libraries.tar.gz -C ~/Library/Application\ Support/com.isaacmarovitz.Whisky/
```

This places the `Libraries/` directory at:

```
~/Library/Application Support/com.isaacmarovitz.Whisky/Libraries/
```

### 3. Verify

Launch Whisky. It should detect version 3.0.0 from the bundled
`WhiskyWineVersion.plist`. If Whisky shows a version mismatch warning, it is
safe to dismiss — the custom build will work normally.

## Directory layout

After extraction, the `Libraries/` directory contains:

```
Libraries/
  WhiskyWineVersion.plist    # Version metadata (3.0.0)
  winetricks                 # Winetricks script
  verbs.txt                  # Winetricks verb list
  DXVK/                      # DXVK binaries (d3d10core, d3d11, dxgi)
    x32/
    x64/
  Wine/
    bin/                     # wine, wineserver, etc.
    lib/
      wine/
        x86_64-unix/         # Unix-side .so modules
        x86_64-windows/      # PE .dll modules
        i386-windows/         # 32-bit PE .dll modules
      external/              # D3DMetal.framework, libd3dshared.dylib (GPTK)
      gstreamer-1.0/         # GStreamer plugins
      libMoltenVK.dylib      # Vulkan (MoltenVK)
      libSDL2-2.0.0.dylib    # SDL2
      ...                    # Other bundled Homebrew dylibs
    share/
      wine/
        mono/                # Wine Mono 10.4.1
```

## Building from source

The CI workflow (`.github/workflows/build.yml`) builds on a `macos-15-intel`
runner. To build locally on an Intel Mac (or under Rosetta):

### Prerequisites

Install build dependencies via Homebrew:

```sh
brew install bison pkg-config mingw-w64 lld \
             freetype gettext gnutls gstreamer sdl2 molten-vk winetricks
```

### Configure and build

```sh
export PATH="$(brew --prefix bison)/bin:$PATH"
export CC=clang CXX=clang++
export CPATH=/usr/local/include
export LIBRARY_PATH=/usr/local/lib
export CFLAGS="-O3"
export CROSSCFLAGS="-O3 -Wno-error=incompatible-pointer-types -Wno-error=int-conversion"
export LDFLAGS="-Wl,-headerpad_max_install_names -Wl,-rpath,@loader_path/../../ -Wl,-rpath,/usr/local/lib"
export MACOSX_DEPLOYMENT_TARGET=14.0
export ac_cv_lib_soname_MoltenVK="libMoltenVK.dylib"
export ac_cv_lib_soname_vulkan=""

mkdir -p build/wine && cd build/wine

../../configure \
  --prefix= \
  --disable-tests \
  --disable-winedbg \
  --enable-archs=i386,x86_64 \
  --with-coreaudio \
  --with-cups \
  --with-freetype \
  --with-gettext \
  --with-gnutls \
  --with-gstreamer \
  --with-mingw \
  --with-opencl \
  --with-pcap \
  --with-pthread \
  --with-sdl \
  --with-unwind \
  --with-vulkan \
  --without-alsa \
  --without-capi \
  --without-dbus \
  --without-fontconfig \
  --without-gettextpo \
  --without-gphoto \
  --without-gssapi \
  --without-krb5 \
  --without-netapi \
  --without-oss \
  --without-pulse \
  --without-sane \
  --without-udev \
  --without-usb \
  --without-v4l2 \
  --without-x

make -j$(sysctl -n hw.ncpu)
make install-lib DESTDIR=$(pwd)/install
```

### Assemble the Libraries package

After building, assemble the final package:

```sh
cd ../..  # back to repo root

mkdir -p Libraries/Wine Libraries/DXVK

# Copy Wine install
cp -a build/wine/install/. Libraries/Wine/
rm -rf Libraries/Wine/share/man

# Copy winetricks
cp -a "$(brew --prefix winetricks)/bin/winetricks" Libraries/
curl -L -o Libraries/verbs.txt \
  https://raw.githubusercontent.com/Winetricks/winetricks/master/files/verbs/all.txt

# Copy DXVK binaries
cp -a DXVK Libraries/

# Bundle external dylibs (GStreamer, MoltenVK, SDL2)
chmod +x .github/dylib_packer.zsh
.github/dylib_packer.zsh

# Install GPTK 3.0
ditto GPTK/redist/lib/ Libraries/Wine/lib/

# Copy version plist
cp -a WhiskyWineVersion.plist Libraries/

# Install Wine Mono
mkdir -p Libraries/Wine/share/wine/mono
curl -L -o mono.tar.xz \
  https://github.com/madewokherd/wine-mono/releases/download/wine-mono-10.4.1/wine-mono-10.4.1-x86.tar.xz
tar -xzf mono.tar.xz -C Libraries/Wine/share/wine/mono

# Create the final tarball
tar -zcf Libraries.tar.gz Libraries
```

Then follow the steps in [Installing from CI artifacts](#2-extract-to-the-whisky-libraries-directory)
to install the resulting `Libraries.tar.gz`.

## GPTK environment variables

GPTK 3.0 exposes environment variables that control D3DMetal and Rosetta
behavior.

| Variable | GPTK default (unset) | Whisky UI | Description |
|---|---|---|---|
| `D3DM_SUPPORT_DXR` | OFF on M1/M2, ON on M3+ | Toggle (off by default, so GPTK's own default applies) | Enable DirectX Raytracing in D3DMetal's DX12 layer |
| `ROSETTA_ADVERTISE_AVX` | OFF | Toggle (off by default) | Advertise AVX/AVX2 cpuid support to translated apps (Apple Silicon, macOS 15+) |
| `D3DM_ENABLE_METALFX` | OFF | Not exposed — requires launching Wine from the terminal | Enable DLSS-to-MetalFX translation (macOS 26 Tahoe required) |

## Enabling MetalFX (DLSS) support

GPTK 3.0 includes experimental DLSS-to-MetalFX translation via
`nvngx-on-metalfx`. This requires macOS 26 Tahoe.

The bundled files are named `nvngx-on-metalfx` and must be renamed and
installed into the Wine prefix to activate:

```sh
LIBRARIES=~/Library/Application\ Support/com.isaacmarovitz.Whisky/Libraries
PREFIX=~/Library/Application\ Support/com.isaacmarovitz.Whisky/Bottles/<bottle>/drive_c

# Rename the .so and .dll in the Wine lib directory
mv "$LIBRARIES/Wine/lib/wine/x86_64-unix/nvngx-on-metalfx.so" \
   "$LIBRARIES/Wine/lib/wine/x86_64-unix/nvngx.so"
mv "$LIBRARIES/Wine/lib/wine/x86_64-windows/nvngx-on-metalfx.dll" \
   "$LIBRARIES/Wine/lib/wine/x86_64-windows/nvngx.dll"

# Copy nvngx.dll and nvapi64.dll into the prefix's system32
cp "$LIBRARIES/Wine/lib/wine/x86_64-windows/nvngx.dll" \
   "$PREFIX/windows/system32/"
cp "$LIBRARIES/Wine/lib/wine/x86_64-windows/nvapi64.dll" \
   "$PREFIX/windows/system32/"
```

Then set `D3DM_ENABLE_METALFX=1` when launching the game.

## Notes

- **Do not pass `--with-opengl` to configure.** Wine 11.0 added an EGL check
  under this flag that produces a hard error on macOS. macOS OpenGL support
  works automatically through `winemac.drv` and `OpenGL.framework`.
- **The build must target x86_64.** GPTK/D3DMetal hooks in winemac.drv are
  guarded by `#if defined(__x86_64__)`. On Apple Silicon, Wine runs under
  Rosetta as an x86_64 process.
- **DXVK dxgi.dll** is built from Wine's own `dlls/dxgi/` source (not from
  DXVK). The DXVK project's `dxgi.dll` does not work on macOS, and Apple's
  D3DMetal `dxgi.dll` is incompatible with DXVK's d3d10core/d3d11. After
  building, the builtin signature (`"Wine builtin DLL"` at PE offset 0x40)
  must be stripped so that Wine's loader treats it as a native DLL. Without
  this, the `WINEDLLOVERRIDES=dxgi=n,b` override is ignored and GPTK's
  dxgi.dll from `lib/wine/x86_64-windows/` is loaded instead. Strip it
  with: `printf '\x00%.0s' {1..17} | dd of=dxgi.dll bs=1 seek=64 count=17
  conv=notrunc`
