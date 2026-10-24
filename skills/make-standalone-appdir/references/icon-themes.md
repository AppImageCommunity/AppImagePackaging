# Bundling icon themes

Step 9 of the assembly in SKILL.md, in full: which theme directories to copy, index.theme, icon caches, and the Adwaita SVG trap. Part of the make-standalone-appdir skill.

### 9. Bundle icon themes

#### hicolor (required)

GTK requires a valid `index.theme` with both `apps` and `actions` contexts:

```sh
mkdir -p AppDir/usr/share/icons/hicolor
if [ -f /usr/share/icons/hicolor/index.theme ]; then
    cp /usr/share/icons/hicolor/index.theme AppDir/usr/share/icons/hicolor/
else
    cat > AppDir/usr/share/icons/hicolor/index.theme <<'EOF'
[Icon Theme]
Name=hicolor
Comment=Fallback icon theme
Directories=16x16/apps,32x32/apps,48x48/apps,scalable/apps,scalable/actions

[16x16/apps]
Size=16
Type=Fixed
Context=Apps

[32x32/apps]
Size=32
Type=Fixed
Context=Apps

[48x48/apps]
Size=48
Type=Fixed
Context=Apps

[scalable/apps]
Size=48
MinSize=16
MaxSize=256
Type=Scalable
Context=Apps

[scalable/actions]
Size=48
MinSize=16
MaxSize=256
Type=Scalable
Context=Actions
EOF
fi

mkdir -p AppDir/usr/share/icons/hicolor/scalable/actions
```

#### Adwaita (for stock GTK icons)

```sh
if [ -d /usr/share/icons/Adwaita ]; then
    mkdir -p AppDir/usr/share/icons
    cp -a /usr/share/icons/Adwaita AppDir/usr/share/icons/
    echo "Adwaita icons bundled"
fi

# Copy standard action icons from Adwaita into hicolor
for icon in list-add edit-copy edit-delete go-up go-down go-bottom \
            changes-prevent view-restore view-reveal view-sidebar-end \
            window-close media-playback-start media-playback-pause; do
    for ext in svg png; do
        src=$(find AppDir/usr/share/icons/Adwaita -name "${icon}-symbolic.${ext}" -type f 2>/dev/null | head -1)
        if [ -n "$src" ]; then
            cp -f "$src" "AppDir/usr/share/icons/hicolor/scalable/actions/" 2>/dev/null || true
        fi
    done
done
```

**Why copy to hicolor:** GTK looks in hicolor first, then falls back to
Adwaita. Standard icons like `list-add-symbolic` live in Adwaita but must
be findable via hicolor for layers panels, menus, etc.
