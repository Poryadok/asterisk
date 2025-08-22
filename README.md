PKG=unity-webgl-support-2023.2.6f1
VER=2023.2.6f1
ARCH=amd64
ARCHIVE=UnitySetup-WebGL-Support-for-Editor-2023.2.6f1.tar.xz
UNITY_ROOT=/opt/unity
WORK=$PWD/$PKG
OUTDIR=$PWD/out

rm -rf "$WORK" "$OUTDIR"
mkdir -p "$WORK/DEBIAN" "$WORK/usr/share/$PKG"

# положим архив внутрь пакета
cp "$ARCHIVE" "$WORK/usr/share/$PKG/"

# control
cat > "$WORK/DEBIAN/control" <<EOF
Package: $PKG
Version: $VER
Section: devel
Priority: optional
Architecture: $ARCH
Maintainer: You <you@example.com>
Description: Unity WebGL Build Support for Unity Editor $VER
EOF

# postinst — сразу с подставленными путями
cat > "$WORK/DEBIAN/postinst" <<EOF
#!/bin/sh
set -e
ARCHIVE="/usr/share/$PKG/$ARCHIVE"
UNITY_ROOT="$UNITY_ROOT"

if [ ! -f "\$ARCHIVE" ]; then
  echo "Archive not found: \$ARCHIVE" >&2
  exit 1
fi
if [ ! -d "\$UNITY_ROOT" ]; then
  echo "Unity root not found: \$UNITY_ROOT" >&2
  exit 1
fi

echo "Extracting \$ARCHIVE to \$UNITY_ROOT ..."
tar -xJf "\$ARCHIVE" -C "\$UNITY_ROOT"
echo "Unity WebGL Support installed."
exit 0
EOF
chmod 0755 "$WORK/DEBIAN/postinst"

# (опционально) postrm на purge
cat > "$WORK/DEBIAN/postrm" <<EOF
#!/bin/sh
set -e
UNITY_ROOT="$UNITY_ROOT"
TARGET="\$UNITY_ROOT/Editor/Data/PlaybackEngines/WebGLSupport"

case "\$1" in
  purge)
    echo "Removing \$TARGET ..."
    rm -rf "\$TARGET"
  ;;
esac
exit 0
EOF
chmod 0755 "$WORK/DEBIAN/postrm"

# сборка
mkdir -p "$OUTDIR"
dpkg-deb --build "$WORK" "$OUTDIR/${PKG}_${VER}_${ARCH}.deb"

# установка
sudo dpkg -i "$OUTDIR/${PKG}_${VER}_${ARCH}.deb"
