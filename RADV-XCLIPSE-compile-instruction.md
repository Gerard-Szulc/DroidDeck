Complete standalone guide, assuming **only** `https://github.com/JimVulkan/radv-xclipse` and a Linux x86_64 machine:

---

# Building the Xclipse 940 RADV driver for DroidDeck — from scratch

**What you get**: `radv-xclipse-driver-Linux.zip` — import it in DroidDeck under *Components → Linux Vulkan drivers → Import*.

**Host needs**: Linux x86_64 with `bash`, `curl`, `tar`, `python3`, `dpkg-deb` (any distro; a bare WSL Ubuntu works). No compilers installed — zig provides them. ~8 GB free disk, ~30–60 min.

```bash
export XCL=~/xcl && mkdir -p "$XCL" && cd "$XCL"
```

## Step 1 — toolchain (zig + meson)

```bash
mkdir -p toolchain/zig toolchain/bin toolchain/reenv
curl -sL https://ziglang.org/download/0.17.0/zig-x86_64-linux-0.17.0.tar.xz \
  | tar -xJ -C toolchain/zig --strip-components=1
python3 -m venv toolchain/reenv
toolchain/reenv/bin/pip install meson ninja mako packaging pyyaml
```

## Step 2 — compiler wrapper scripts (7 tiny files)

Zig can't take `-Wl,--version-script FILE` in two args, and the fork needs a *host* compiler for the wayland protocol generators — these wrappers fix both:

```bash
cat > toolchain/bin/zig-cc <<'EOF'
#!/bin/bash
args=()
prev_vs=0
for a in "$@"; do
  if [ "$prev_vs" = "1" ]; then
    args+=("-Wl,--version-script=$a"); prev_vs=0; continue
  fi
  case "$a" in
    -Wl,--version-script) prev_vs=1 ;;
    *) args+=("$a") ;;
  esac
done
exec $XCL/toolchain/zig/zig cc -target aarch64-linux-gnu "${args[@]}"
EOF
sed 's| zig cc -target aarch64-linux-gnu| zig c++ -target aarch64-linux-gnu|' \
    toolchain/bin/zig-cc > toolchain/bin/zig-c++
printf '#!/bin/sh\nexec %s/toolchain/zig/zig ar "$@"\n' "$XCL" > toolchain/bin/zig-ar
printf '#!/bin/sh\nexec %s/toolchain/zig/zig cc -target x86_64-linux-musl "$@"\n' "$XCL" > toolchain/bin/zig-cc-native
printf '#!/bin/sh\nexec %s/toolchain/zig/zig c++ -target x86_64-linux-musl "$@"\n' "$XCL" > toolchain/bin/zig-c++-native
cat > toolchain/bin/pkg-config <<'EOF'
#!/bin/sh
export LD_LIBRARY_PATH=$XCL/wsi-deps/host-amd64/usr/lib/x86_64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
exec $XCL/wsi-deps/host-amd64/usr/bin/pkgconf "$@"
EOF
cat > toolchain/bin/pkg-config-native <<'EOF'
#!/bin/sh
export LD_LIBRARY_PATH=$XCL/wsi-deps/host-amd64/usr/lib/x86_64-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
export PKG_CONFIG_LIBDIR=$XCL/wsi-deps/host-amd64/usr/lib/x86_64-linux-gnu/pkgconfig:$XCL/wsi-deps/host-amd64/usr/share/pkgconfig
exec $XCL/wsi-deps/host-amd64/usr/bin/pkgconf "$@"
EOF
chmod +x toolchain/bin/*
```

## Step 3 — Android stubs (the fork includes android headers unconditionally)

```bash
mkdir -p stubs/android stubs/sys
cat > stubs/android/log.h <<'EOF'
#pragma once
#include <stdarg.h>
enum {
   ANDROID_LOG_VERBOSE = 2, ANDROID_LOG_DEBUG = 3, ANDROID_LOG_INFO = 4,
   ANDROID_LOG_WARN = 5, ANDROID_LOG_ERROR = 6, ANDROID_LOG_FATAL = 7,
};
static inline int __android_log_print(int prio, const char *tag, const char *fmt, ...)
{ (void)prio; (void)tag; (void)fmt; return 0; }
static inline int __android_log_vprint(int prio, const char *tag, const char *fmt, va_list ap)
{ (void)prio; (void)tag; (void)fmt; (void)ap; return 0; }
EOF
cat > stubs/sys/system_properties.h <<'EOF'
#pragma once
#define PROP_VALUE_MAX 92
static inline int __system_property_get(const char *name, char *value)
{ (void)name; if (value) value[0] = '\0'; return 0; }
EOF
```

## Step 4 — clone the fork, restore the files it pruned

The fork ships a vulkan-only tree. The wayland/x11 WSI needs `src/x11`, `src/loader`, `include/pci_ids`, some `drm-uapi` and `vulkan` headers — restore them from upstream Mesa:

```bash
git clone https://github.com/JimVulkan/radv-xclipse.git
cd radv-xclipse
curl -sL https://gitlab.freedesktop.org/mesa/mesa/-/archive/main/mesa-main.tar.gz -o /tmp/mesa-main.tar.gz
tar -xzf /tmp/mesa-main.tar.gz mesa-main/src/x11 mesa-main/src/loader \
    mesa-main/include/pci_ids mesa-main/include/drm-uapi mesa-main/include/vulkan
cp -rn mesa-main/src/x11 mesa-main/src/loader src/
mkdir -p include/pci_ids && cp -rn mesa-main/include/pci_ids/. include/pci_ids/
for f in mesa-main/include/drm-uapi/*.h; do
  b=$(basename "$f"); [ -f "include/drm-uapi/$b" ] || cp "$f" include/drm-uapi/
done
for f in mesa-main/include/vulkan/*.h; do
  b=$(basename "$f"); [ -f "include/vulkan/$b" ] || cp "$f" include/vulkan/
done
rm -rf mesa-main /tmp/mesa-main.tar.gz
( cd subprojects/libdrm-* && ln -sf include/drm drm-uapi )
cd ..
```

## Step 5 — Debian trixie sysroot (arm64) + host tools (amd64)

```bash
mkdir -p wsi-deps/sysroot-arm64 wsi-deps/host-amd64
DEB=http://ftp.us.debian.org/debian
arm64() { local f b; for f in "$@"; do b=${f##*/}; \
  [ -f "wsi-deps/$b" ] || curl -sL "$DEB/$f" -o "wsi-deps/$b"; \
  dpkg-deb -x "wsi-deps/$b" wsi-deps/sysroot-arm64; done; }
host64() { local f b; for f in "$@"; do b=${f##*/}; \
  [ -f "wsi-deps/$b" ] || curl -sL "$DEB/$f" -o "wsi-deps/$b"; \
  dpkg-deb -x "wsi-deps/$b" wsi-deps/host-amd64; done; }

arm64 \
  pool/main/libf/libffi/libffi8_3.4.8-2_arm64.deb \
  pool/main/libf/libffi/libffi-dev_3.4.8-2_arm64.deb \
  pool/main/w/wayland/libwayland-client0_1.23.1-3_arm64.deb \
  pool/main/w/wayland/libwayland-server0_1.23.1-3_arm64.deb \
  pool/main/w/wayland/libwayland-dev_1.23.1-3_arm64.deb \
  pool/main/libx/libx11/libx11-6_1.8.12-1_arm64.deb \
  pool/main/libx/libx11/libx11-dev_1.8.12-1_arm64.deb \
  pool/main/libx/libx11/libx11-xcb1_1.8.12-1_arm64.deb \
  pool/main/libx/libx11/libx11-xcb-dev_1.8.12-1_arm64.deb \
  pool/main/libx/libxau/libxau-dev_1.0.11-1_arm64.deb \
  pool/main/libx/libxdmcp/libxdmcp-dev_1.1.5-1_arm64.deb \
  pool/main/libx/libxext/libxext6_1.3.4-1+b3_arm64.deb \
  pool/main/libx/libxext/libxext-dev_1.3.4-1+b3_arm64.deb \
  pool/main/libx/libxfixes/libxfixes3_6.0.0-2+b4_arm64.deb \
  pool/main/libx/libxfixes/libxfixes-dev_6.0.0-2+b4_arm64.deb \
  pool/main/libx/libxinerama/libxinerama1_1.1.4-3+b4_arm64.deb \
  pool/main/libx/libxinerama/libxinerama-dev_1.1.4-3+b4_arm64.deb \
  pool/main/libx/libxrandr/libxrandr2_1.5.4-1+b3_arm64.deb \
  pool/main/libx/libxrandr/libxrandr-dev_1.5.4-1+b3_arm64.deb \
  pool/main/libx/libxrender/libxrender1_0.9.12-1_arm64.deb \
  pool/main/libx/libxrender/libxrender-dev_0.9.12-1_arm64.deb \
  pool/main/libx/libxshmfence/libxshmfence1_1.3.3-1_arm64.deb \
  pool/main/libx/libxshmfence/libxshmfence-dev_1.3.3-1_arm64.deb \
  pool/main/libx/libxcb/libxcb1_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb1-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-dri3-0_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-dri3-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-present0_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-present-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-randr0_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-randr0-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-render0-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-shape0-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-shm0_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-shm0-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-sync1_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-sync-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-xfixes0_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-xfixes0-dev_1.17.0-2+b1_arm64.deb \
  pool/main/libx/libxcb/libxcb-xkb-dev_1.17.0-2+b1_arm64.deb \
  pool/main/x/xcb-util-keysyms/libxcb-keysyms1_0.4.1-1_arm64.deb \
  pool/main/x/xcb-util-keysyms/libxcb-keysyms1-dev_0.4.1-1_arm64.deb \
  pool/main/x/xorgproto/x11proto-dev_2024.1-1_all.deb \
  pool/main/w/wayland-protocols/wayland-protocols_1.44-1_all.deb

host64 \
  pool/main/p/pkgconf/libpkgconf3_1.8.1-4_amd64.deb \
  pool/main/p/pkgconf/pkgconf-bin_1.8.1-4_amd64.deb \
  pool/main/w/wayland/libwayland-bin_1.23.1-3_amd64.deb \
  pool/main/libx/libxml2/libxml2_2.12.7+dfsg+really2.9.14-2.1+deb13u3_amd64.deb \
  pool/main/x/xz-utils/liblzma5_5.8.1-1+deb13u1_amd64.deb \
  pool/main/g/glslang/glslang-tools_15.1.0+1.4.309.0-1_amd64.deb \
  pool/main/w/wayland-protocols/wayland-protocols_1.44-1_all.deb
```

## Step 6 — fix the sysroot (three non-obvious fixes)

```bash
SYS=$XCL/wsi-deps/sysroot-arm64
HOST=$XCL/wsi-deps/host-amd64

# 1. point every .pc at the extracted tree
find "$SYS/usr/lib/aarch64-linux-gnu/pkgconfig" "$SYS/usr/share/pkgconfig" \
     -name '*.pc' -exec sed -i "s|^prefix=/usr\$|prefix=$SYS/usr|" {} +
sed -i "s|^pkgdatadir=.*|pkgdatadir=$SYS/usr/share/wayland-protocols|" \
    "$SYS/usr/share/pkgconfig/wayland-protocols.pc"
sed -i "s|^prefix=/usr\$|prefix=$HOST/usr|; s|^pkgdatadir=.*|pkgdatadir=$HOST/usr/share/wayland-protocols|" \
    "$HOST/usr/share/pkgconfig/wayland-protocols.pc"

# 2. trixie's wayland .pc lacks the scanner variable; meson then looks at
#    /usr/bin/wayland-scanner and fails. Provide our own.
cat > "$SYS/usr/lib/aarch64-linux-gnu/pkgconfig/wayland-scanner.pc" <<EOF
prefix=$HOST/usr
bindir=\${prefix}/bin

Name: wayland-scanner
Description: wayland-scanner
Version: 1.23.1
wayland_scanner=\${bindir}/wayland-scanner
EOF
cp "$SYS/usr/lib/aarch64-linux-gnu/pkgconfig/wayland-scanner.pc" "$SYS/usr/share/pkgconfig/"
cp "$SYS/usr/lib/aarch64-linux-gnu/pkgconfig/wayland-scanner.pc" "$HOST/usr/share/pkgconfig/"

# 3. move every static .a away: linking them needs trixie-glibc symbols
#    (__isoc23_strtol) that zig's glibc doesn't have. The .so files link
#    cleanly and the Arch runtime provides them at run time.
mkdir -p "$SYS/usr/lib/aarch64-linux-gnu/.static-bak"
cd "$SYS/usr/lib/aarch64-linux-gnu"
for a in *.a; do [ -e "$a" ] && mv -n "$a" .static-bak/; done
cd "$XCL"
```

## Step 7 — meson machine files

```bash
cat > glibc-cross.ini <<EOF
[binaries]
c = '$XCL/toolchain/bin/zig-cc'
cpp = '$XCL/toolchain/bin/zig-c++'
ar = '$XCL/toolchain/bin/zig-ar'
pkg-config = '$XCL/toolchain/bin/pkg-config'

[host_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'aarch64'
endian = 'little'
EOF
cat > native.ini <<EOF
[binaries]
c = '$XCL/toolchain/bin/zig-cc-native'
cpp = '$XCL/toolchain/bin/zig-c++-native'
ar = '$XCL/toolchain/bin/zig-ar'
pkg-config = '$XCL/toolchain/bin/pkg-config-native'
EOF
```

(`pkg-config` **must** be in the cross file — meson ignores the native file for it in cross builds.)

## Step 8 — build

```bash
cd $XCL/radv-xclipse
export PATH="$XCL/toolchain/reenv/bin:$XCL/toolchain/bin:$XCL/wsi-deps/host-amd64/usr/bin:$PATH"
export PKG_CONFIG_LIBDIR="$XCL/wsi-deps/sysroot-arm64/usr/lib/aarch64-linux-gnu/pkgconfig:$XCL/wsi-deps/sysroot-arm64/usr/share/pkgconfig"
export LD_LIBRARY_PATH="$XCL/wsi-deps/host-amd64/usr/lib/x86_64-linux-gnu"

rm -rf build-wsi
meson setup build-wsi \
  --cross-file "$XCL/glibc-cross.ini" --native-file "$XCL/native.ini" \
  -Dbuildtype=release -Db_ndebug=true \
  -Dplatforms=wayland,x11 -Dvulkan-drivers=amd -Dgallium-drivers= -Dvulkan-layers= -Dtools= \
  -Dllvm=disabled -Dzstd=disabled -Dlmsensors=disabled -Dperfetto=false \
  -Dglx=disabled -Dgbm=disabled -Degl=disabled -Dgles1=disabled -Dgles2=disabled \
  -Dallow-fallback-for=libdrm,perfetto --force-fallback-for=expat,libdrm,zlib \
  -Dlibdrm:default_library=static -Dexpat:default_library=static -Dzlib:default_library=static \
  -Dc_args=-I"$XCL/stubs" -Dcpp_args=-I"$XCL/stubs"

ninja -C build-wsi src/amd/vulkan/libvulkan_radeon.so
```

(If a dependency error appears, the answer is almost always "add that package's arm64 `.deb` to step 5" — the `.pc` error names the package.)

## Step 9 — strip and package

```bash
# strip with any aarch64 stripper; on Debian/Ubuntu:  apt install binutils-aarch64-linux-gnu
# then:  aarch64-linux-gnu-strip --strip-unneeded build-wsi/src/amd/vulkan/libvulkan_radeon.so
# (unstripped also works, it's just ~113 MB instead of ~19 MB)

mkdir -p driver-zip
cat > driver-zip/meta.json <<'EOF'
{
  "name": "radv-xclipse-26.3.0",
  "driverVersion": "26.3.0-xcl940",
  "minGlibc": "2.34",
  "kind": "linux-vulkan-icd"
}
EOF
python3 - <<'EOF'
import zipfile
z = zipfile.ZipFile('radv-xclipse-driver-Linux.zip', 'w', zipfile.ZIP_DEFLATED, compresslevel=9)
z.write('radv-xclipse/build-wsi/src/amd/vulkan/libvulkan_radeon.so', 'libvulkan_radeon.so')
z.write('driver-zip/meta.json', 'meta.json')
z.close()
EOF
```

## Step 10 — install on the phone

```bash
adb push radv-xclipse-driver-Linux.zip /sdcard/Download/
```
Then in DroidDeck: **Components → Linux Vulkan drivers → Import** → pick the zip.

---

**Sanity check**: `readelf -d` on the built `.so` should list `libwayland-client.so.0`, `libxcb.so.1`, `libxcb-dri3.so.0`, `libxcb-present.so.0`, `libxshmfence.so.1`, and **no** `libdrm` (statically linked). If a Debian `.deb` 404s in the future, grab the current version from `http://ftp.us.debian.org/debian/dists/trixie/main/binary-arm64/Packages.gz`.