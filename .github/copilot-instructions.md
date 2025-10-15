## Kurz: Zweck dieses Dokuments
Konzentrierte, direkt nutzbare Instruktionen für AI-Coding-Agenten, damit sie in der WSLg-Codebasis produktiv arbeiten können. Kurze Architektur, konkrete Build-/Debug-Beispiele, lokale Einschränkungen und Pfade zu relevanten Dateien.

## Kern-Architektur (kompakt)
- WSLg remoted Linux-GUIs (Wayland/X11) aus einer Linux "system distro" nach Windows über RDP.
- Wichtige Komponenten:
  - `WSLGd/` — Start/Monitor von Weston (Compositor), PulseAudio und RDP-Verbindung (mstsc auf Host).
  - `WSLDVCPlugin/` — Windows-seitiger mstsc-Plugin (Start-Menu-Integration).
  - `rdpapplist/` — Plugin/Server-Seite, die Desktop-Dateien (apps) auflistet und dem Host sendet.
  - `vendor/` — erwartet Forks (FreeRDP, Weston, PulseAudio, Mesa, DirectX-Headers) auf Branch `working`.

## Schnellstart für Entwickler (aus den Repo-Inhalten)
- Lokales Image / VHD bauen (Linux/WSL2 mit Docker):

```bash
# Voraussetzungen: Docker, go (für tar2ext4)
git clone https://github.com/microsoft/wslg wslg
cd ..
# Klone Mirrors in wslg/vendor und checke 'working' aus (siehe CONTRIBUTING.md)
git clone https://github.com/microsoft/FreeRDP-mirror wslg/vendor/FreeRDP -b working
git clone https://github.com/microsoft/weston-mirror wslg/vendor/weston -b working
git clone https://github.com/microsoft/pulseaudio-mirror wslg/vendor/pulseaudio -b working
# Build Docker image und export
sudo docker build -f ./wslg/Dockerfile -t system-distro-x64 ./wslg --build-arg WSLG_ARCH=x86_64
sudo docker export `sudo docker create system-distro-x64` > system_x64.tar
cd hcsshim/cmd/tar2ext4
go run tar2ext4.go -vhd -i ../../../system_x64.tar -o ../../../system.vhd
```

Hinweis: Die CI (`azure-pipelines.yml`) zeigt ausführlichere Schritte (mesa/DirectX-Headers herunterladen, buildx für ARM, artefakte publish). Verwende CI als Referenz für reproduzierbare Schritte.

## Was lokal in diesem Devcontainer möglich ist
- Lesbare/kompilierbare Linux-Komponenten: `WSLGd/` (Makefile), `rdpapplist/server` (meson), `rdpapplist/rdpapplist_common.c` sind buildbar auf Linux.
- Windows-spezifisch: `WSLDVCPlugin/` erfordert Visual Studio / MSBuild und `mstsc.exe` (Host) — diese Schritte können nicht vollständig in diesem Linux-Container ausgeführt werden.
- System-Distro Erzeugung (VHD) ist in Linux möglich (Docker + tar2ext4). Start/Integration in Windows muss auf einem Windows-Host erfolgen.

### Lokaler Quickstart (devcontainer)
- Schneller Build von `WSLGd` (direkt im Container, clang++/make):

```bash
# im Repo-Root
cd WSLGd
make
# erzeugt ./WSLGd
```

- `rdpapplist/server` verwendet Meson/Ninja; `meson` ist in diesem Devcontainer nicht vorinstalliert. Falls benötigt:

```bash
sudo apt update && sudo apt install -y python3-pip python3-venv ninja-build
pip3 install --user meson
export PATH="$HOME/.local/bin:$PATH"
cd rdpapplist/server
meson setup builddir
ninja -C builddir
```

Hinweis: Meson/Ninja Schritte sind optional für tiefergehende Änderungen an `rdpapplist`.

## Wichtige Dateien / Einstiegspunkte (konkret)
- Build / Image: `Dockerfile`, `container/build.sh`, `config/BUILD.md`, `CONTRIBUTING.md`
- Daemon: `WSLGd/main.cpp`, `WSLGd/Makefile`
- RDP-App-Listing: `rdpapplist/server/rdpapplist_main.c`, `rdpapplist/rdpapplist_common.c`, `rdpapplist/server/meson.build`
- Windows plugin: `WSLDVCPlugin/WSLDVCPlugin.sln`, `WSLDVCPlugin/README` (siehe Dateien im Ordner)
- CI-Referenz: `azure-pipelines.yml` (zeigt cross-arch Builds + artifact handling)

## Konkrete Hinweise für AI-Agenten / Änderungen
- Verwende immer `vendor/*` Forks auf Branch `working` für lokale builds — CI setzt das voraus.
- Minimale, nicht-invasive Änderungen: Wenn du Änderungen an Weston/FreeRDP-spezifischen Verhalten machst, dokumentiere die Änderung im `rdpapplist/` und aktualisiere `azure-pipelines.yml` wenn neue build-artefakte nötig sind.
- Config-Änderungen für Tests: `config/weston.ini` und `config/local.conf` werden von `WSLGd`/system-distro erwartet. Nutze diese Dateien zum Reproduzieren von Laufzeit-Setups.

## Debugging-Tipps (konkret)
- Im System-Distro prüfen: `ps -ax | grep weston` und `cat /mnt/wslg/weston.log` (siehe README/CONTRIBUTING Beispiels-Ausgabe).
- Logs in `WSLGd`: `WSLGd` schreibt Logs (siehe `WSLGd/main.cpp`), starte lokal mit `make` und prüfe stdout/stderr.
- Startmenu/Integration-Tests: müssen auf Windows-Host geprüft werden. Verwende die Registry-Key-Beispiele in `CONTRIBUTING.md` für private `WSLDVCPlugin`-Loading.

## Einschränkungen und Annahmen
- Annahme: Entwickler haben Zugang zu Docker und können große Repo-Klone (Mesa, DirectX-Headers) herunterladen.
- Einschränkung: Vollständige End-to-End-Validierung (Start-Menu-Integration, mstsc) benötigt Windows mit RDP-Client (`mstsc.exe`) und Administratorrechte.

Falls du möchtest, kann ich:
- a) Diese Datei weiter kürzen/übersetzen, oder
- b) Schritt-für-Schritt-Skripte erstellen, um die Linux-Builds hier im Container zu starten (meson/Makefile builds), oder
- c) Einen kurzen Check-Run ausführen (ls/meson --version/make -n) um die lokale Build-Fähigkeit zu verifizieren.

Bitte sag mir, welche Option du bevorzugst — ich mache anschließend den gewählten Schritt und berichte Ergebnis.
