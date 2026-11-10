# PZ Optimization on Build 42.22.2, Windows (unofficial)

This fork builds [xD3I's PZ_Optimization](https://github.com/xD3I/PZ_Optimization) for **Project Zomboid 42.22.2**
(revision `3d39148846`) on **Windows**, from source. It is not an official release: problems with it are this fork's,
not upstream's. Upstream has no 42.22 release yet.

## Why a fork, and why the extra launch option

- **42.22 signs `projectzomboid.jar`.** PZ Optimization used to work by putting edited game classes in the game folder,
  which the game loads before its jar. Java refuses unsigned classes in the packages of a signed jar, so that no longer
  works for anyone on 42.22.
- **Upstream's answer is a Java agent**, on its unreleased `integrity-agent` branch (written for 42.22.1): the classes
  move to `pzopt/overrides/` and `-javaagent:pzopt/agent/pzopt-agent.jar` in `ProjectZomboid64.json` hands them to the
  JVM as the game's classes load. Read `docs/findings-signed-jar-2026-11-04.md` for the design. This fork is that
  branch plus:
  - the 42.22.2 revision in `MainScreenState` (the only stock change from 42.22.1 to 42.22.2 under the overrides:
    `build.sh`'s bytecode audit compares 9,539 unedited methods with the 42.22.2 jar, 0 mismatches);
  - Windows fixes to the build scripts (the source list goes to `javac` in a file, the audit reads Windows paths);
  - `src/launch/pzopt-launch.cmd`, the launch fix below.
- **On Windows the agent does not start from Steam.** The JVM loads agent libraries before `java.dll`, and the agent's
  `instrument.dll` needs `java.dll` and `jli.dll` from `jre64\bin`. `java.exe` finds them because it sits in that
  folder; Steam starts `ProjectZomboid64.exe` from the game folder, where Windows does not look. The game then closes
  at once with nothing in `console.txt` (the error, seen only when the exe runs from a console, is
  `Could not find agent library instrument on the library path, with error: Can't find dependent libraries`).
  Copying the DLLs into the game folder would load `java.dll` twice, and the game's Java has no `jdk.attach` module,
  so the agent cannot be attached later. The fix is a launch option that runs the game through `pzopt-launch.cmd`,
  which puts the game's own `jre64\bin` first on `PATH` for that launch only.

## Before you start

- **The game's integrity report shows it.** 42.22 added `zombie.core.IntegrityReport`: it checks the jar's signature
  and lists class path folders and agents in `console.txt` / `version.txt` (`classpath=...,agents=1` with this
  installed). Nothing hides from it and nothing touches it. Its source carries The Indie Stone's warning that tampering
  with that code means permission to ban you from the Workshop. Upstream holds this approach until The Indie Stone
  answers. Play with it on your own install; do not upload builds of it.
- **Multiplayer is untested.** A server should not refuse it: the login compares only the game version ("42.22.2"),
  not the integrity report.
- **Other Java mods with loose class files in the game folder break** on the signed jar, with or without this.
- **The next game update turns it off** (the overrides refuse any other revision), and it needs a rebuild.

## Build

You need Git for Windows (Git Bash), a JDK 25 or newer, and Python 3.

1. **JDK 25**: unzip a Windows x64 JDK 25 (for example Temurin, from adoptium.net) anywhere, e.g. `D:\tools\jdk-25`.
2. **`python3` in Git Bash**: the Microsoft Store `python3` stub does not run scripts. Make a shim that calls the `py`
   launcher:
   ```sh
   mkdir -p ~/bin && printf '#!/bin/sh\nexec py -3 "$@"\n' > ~/bin/python3 && chmod +x ~/bin/python3
   ```
3. **Environment**, in every Git Bash window you build from (adjust the paths):
   ```sh
   export PATH="$HOME/bin:/d/tools/jdk-25/bin:$PATH"
   export PZ_ROOT="/c/Program Files (x86)/Steam/steamapps/common/ProjectZomboid"
   export PZ_DIR="$PZ_ROOT"
   export ZOMBOID="$HOME/Zomboid"
   ```
4. **Clone and build** (a short path is safest):
   ```sh
   git clone -b port-42.22.2-agent https://github.com/tieeeeen1994/PZ_Optimization.git /d/PZ_Optimization
   cd /d/PZ_Optimization
   scripts/build.sh
   ```
   It takes a few minutes and ends with `built N class files into build/classes for game revision 3d39148846`
   and `projectzomboid.jar is signed: the overrides load through the agent`. It refuses to finish on any mismatch.
   `scripts/test.sh` runs the unit tests; on Windows the three `Ground*Test`s fail because they compile a Linux C tool.

## Install

Close the game, then:

```sh
scripts/pzopt.sh install
scripts/pzopt.sh status    # installed: yes, nothing MISSING or MODIFIED
```

This copies the build into the game folder (listed in `pzopt-installed.txt`), adds the `-javaagent` line to
`ProjectZomboid64.json` and leaves `projectzomboid.jar` untouched. Back up `ProjectZomboid64.json` first if you have
edited it: PZ Optimization also edits its GC flags at start (and undoes that on uninstall).

Then in Steam: **Project Zomboid > Properties > General > Launch Options**:

```
"C:\Windows\System32\conhost.exe" --headless "C:\Program Files (x86)\Steam\steamapps\common\ProjectZomboid\pzopt-launch.cmd" %command%
```

Use your own game folder in the middle. Keep any options you already had after `%command%` (for example
`... %command% -debug`). Both paths must be full paths: Steam does not search Windows' folders for the first word
(a bare `conhost.exe` fails with "Missing game executable"). `--headless` hides the console window; without the
`conhost` part the game works too, with a console window open while you play.

**Check it loaded**: the main menu has a **PZ OPTIMIZATION** item, and `%USERPROFILE%\Zomboid\console.txt` has
`classpath=...,agents=1` and many `[pzopt] loaded override ... active` lines.

## Uninstall

Main menu > PZ OPTIMIZATION > Uninstall PZ Optimization, or double-click `Uninstall-PZ-Optimization.cmd` in the game
folder, or `scripts/pzopt.sh uninstall`. Each removes the files and the `-javaagent` line. **Then set the Steam launch
options back** to what you had: the uninstall removes `pzopt-launch.cmd`, and Steam cannot start a missing file.

If the game does not start at all and you cannot reach the menu: run the uninstaller from the game folder, clear the
launch options, and start the game again.
