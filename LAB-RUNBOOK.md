# Lab-Runbook: Tahoe auf iMac15,1 (tahoe-experiment)

Ziel: macOS Tahoe 26 auf iMac 5K Late 2014 / Mid 2015 (iMac15,1, R9 M290X)
mit Root-Patches der Legacy-GCN-Familie (Metal 31001). Testgerät ist
verzichtbar; Fokusgerät iMac17,1 wird erst nach Erfolg angefasst.

## Stand des Branches

- `main` (2.5.0) + `origin/macos-next` gemerged: Tahoe-Installer freigeschaltet,
  USB-Maps 26, FileVault-Fix, Modern Audio, OpenCore 1.0.5 rolling.
- PatcherSupportPkg auf 1.9.6 gehalten (neuer als macos-next-Pin 1.9.5).
- Root-Patch-Ceiling auf Tahoe angehoben (`detect.py`, Commit 15f40d36a).
- Payloads sind weiterhin Monterey-Ära ("12.5"-Quellen in `amd_legacy_gcn.py`)
  aus Universal-Binaries 1.9.6 — Tahoe-Validierung ist genau das Experiment.

## Phase 0: Bootstrap & Fusion-Split (Maschine ist leer)

Kontext: iMac15,1 offiziell bis Monterey; Fusion Drive (Blade-SSD ~107-121 GB
+ HDD) muss getrennt werden — Tahoe unterstuetzt Fusion Drives nicht mehr.

1. Auf einem beliebigen Mac mit diesem Branch: OCLP starten, Sequoia-Installer
   laden, USB-Stick (>= 32 GB) erstellen. In den Settings **Target Model:
   iMac15,1** setzen und OpenCore auf die EFI des Sticks installieren.
2. iMac mit Alt-Taste booten -> "EFI Boot" -> OpenCore-Picker -> Installer.
3. Im Installer: Dienstprogramme -> **Terminal** (nicht Festplattendienst-
   programm — das will den Fusion Drive "reparieren"):
   - `diskutil list` (Blade-SSD meist disk0, HDD disk1)
   - APFS-Fusion: `diskutil apfs deleteContainer <synth. Container>`
   - CoreStorage:  `diskutil cs list` + `diskutil cs delete <LVG-UUID>`
   - `diskutil eraseDisk APFS Macintosh GPT disk0` (SSD, Systemplatte)
   - `diskutil eraseDisk APFS Daten GPT disk1` (HDD, Datenplatte)
   - NIEMALS `diskutil resetFusion` (stellt den Fusion Drive wieder her!)
4. Sequoia auf "Macintosh" installieren, OCLP-Post-Install: OpenCore auf
   interne ESP, Root-Patches (Sequoia = erprobter Legacy-GCN-Pfad, dient als
   Referenz, dass die Hardware sauber patcht). SSH sofort aktivieren.
5. Tahoe-Testvolume: zweites APFS-Volume im selben SSD-Container anlegen
   (teilt freien Platz). Bei Platznot (107 GB fuer zwei Systeme ist knapp):
   externe USB-3-SSD statt HDD — die HDD ist als Systemplatte zu langsam.

## Vorbereitung (am Schreibtisch)

1. Externe SSD oder eigene APFS-Partition am Testgerät, >= 80 GB.
2. Vollbackup des Testgeräts entbehrlich, aber NVRAM-Reset-Kombi und
   Sequoia-Recovery-Weg parat haben.
3. Zweitrechner mit SSH auf das Testgerät (Systemeinstellungen > Freigaben >
   Entfernte Anmeldung aktivieren, BEVOR gepatcht wird) — Pflicht, um bei
   WindowServer-Crash-Loops Logs zu ziehen.

## Ablauf am Testgerät

1. Repo + Branch auf das Testgerät bringen, venv wie hier:
   `python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`
2. OCLP starten: `.venv/bin/python OpenCore-Patcher-GUI.command`
3. Tahoe-Installer laden (Download macOS Installer -> 26.x), USB-Stick bauen.
4. OpenCore-EFI bauen + auf USB/ESP installieren, von OpenCore booten.
5. Tahoe auf die Testpartition installieren (mehrere Reboots, immer via
   OpenCore-Picker booten).
6. Erster Boot: unbeschleunigt = erwartet. SSH-Zugang verifizieren.
7. Root-Patches anwenden (Post-Install Root Patch). Erwartet: KDK-Download
   fuer 26.x-Build, dann Patch + Snapshot-Bless.
8. Reboot und beobachten.

## Ausgaenge & Reaktion

- **UI beschleunigt, glitcht evtl.** -> Jackpot. Screenshots,
  `system_profiler SPDisplaysDataType`, Metal-Check, Bericht an
  dortania#1167.
- **WindowServer-Crash-Loop** -> per SSH: `log show --last 5m --predicate
  'process == "WindowServer"'`, Crash-Reports aus
  `/Library/Logs/DiagnosticReports`, dann Revert:
  `Last Sequoia snapshot` via OCLP-Revert oder Boot in Recovery.
- **Kernel Panic beim Kext-Load** -> Panic-Log sichern (NVRAM/Reboot-Report),
  dokumentieren, Revert. Naechster Versuch erst mit Upstream-Payloads.

## Revert-Wege (vom harmlosesten zum haertesten)

1. OCLP "Revert Root Patches" (bootet letzten unpatched Snapshot).
2. Recovery -> Terminal -> `bless --mount /Volumes/<Tahoe> --bootefi` bzw.
   Snapshot-Auswahl beim Start (Volume gedrueckt halten im Picker).
3. Testpartition plattmachen — dafuer ist sie da.

## Notizen

- `touch ~/.dortania_developer` schaltet in Upstream-Builds (Nightlies!) das
  OS-Gate frei — nuetzlich, um parallel offizielle Nightlies zu testen.
- 2-GB-VRAM-Geraet: nach Erfolg "Transparenz reduzieren" in den
  Bedienungshilfen aktivieren und Performance vergleichen.
- Upstream-Payload-Watch: dortania/PatcherSupportPkg Releases > 1.9.6 und
  OCLP-Nightlies (GitHub Actions) — sobald dort Legacy-GCN-Tahoe-Material
  auftaucht, PSP-Version in `constants.py` hochziehen und neu testen.
