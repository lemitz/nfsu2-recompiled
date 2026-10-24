# Build an Android APK locally from your own game copy

This repository deliberately omits `SPEED2.EXE`, game assets, generated C and
APK binaries. The compiled `libnfsu2_android.so` **contains code lifted from the
executable** even though the APK does not contain an EXE file. Keep resulting
APKs private; a public source archive can include only the tooling changes.

On Windows, install Git, JDK 21, Python 3.10+, Android SDK Platform 35, NDK
28.2.13676358 and SDK CMake 3.30.5. About 32 GiB of RAM is recommended for
compiling the generated game. The commands below assume PowerShell and that
your legitimately obtained game is installed in `C:\Games\NFSU2`; adjust paths
for your own installation. The tested build had a different EXE hash from the
upstream README but matching hook addresses; other EXE builds may need a
separate hook table as explained in the upstream README.

```powershell
git clone --recurse-submodules https://github.com/lemitz/nfsu2-recompiled.git
cd nfsu2-recompiled
$Game = 'C:\Games\NFSU2'
$env:ANDROID_HOME = 'Z:\SDK'  # Replace with your Android SDK location.
$env:ANDROID_SDK_ROOT = $env:ANDROID_HOME
$env:Path = "$env:ANDROID_HOME\emulator\lib64\vulkan;$env:ANDROID_HOME\ndk\28.2.13676358\shader-tools\windows-x86_64;$env:Path"
python -m pip install capstone pefile unicorn keystone-engine

git clone https://github.com/sp00nznet/pcrecomp.git ../pcrecomp
git -C ../pcrecomp checkout 35548b8
$patches = Get-ChildItem patches/pcrecomp/*.patch | Sort-Object Name | ForEach-Object FullName
git -C ../pcrecomp am $patches

New-Item -ItemType Directory -Force ../reference | Out-Null
git clone --branch native-1.9.2b https://github.com/Joshua-Ashton/dxvk-native.git ../reference/dxvk-native
python ports/dxvk_android/prepare.py ../reference/dxvk-native build/dxvk-source
python ports/dxvk_android/shaders.py build/dxvk-source

New-Item -ItemType Directory -Force work | Out-Null
python scripts/normalize_exe.py "$Game\SPEED2.EXE" work/SPEED2.analysis.exe
python ../pcrecomp/tools/cpp/vtable_scan.py work/SPEED2.analysis.exe --seeds work/vtable_seeds.json
python ../pcrecomp/tools/cpp/rtti.py work/SPEED2.analysis.exe --seeds work/rtti_seeds.json
@'
import json
from pathlib import Path
folder = Path('work')
seeds = sorted({item['address'] for name in ('vtable_seeds', 'rtti_seeds')
                for item in json.loads((folder / (name + '.json')).read_text())})
(folder / 'seeds_all.json').write_text(json.dumps(seeds))
'@ | python -
python scripts/run_lift.py catalog work/SPEED2.analysis.exe work/seeds_all.json work/catalog.json
python scripts/run_lift.py lift work/SPEED2.analysis.exe work/catalog.json src/recomp/gen

cd android
.\gradlew.bat --no-daemon assembleDebug --max-workers=4
```

The locally signed debug APK is at `android/app/build/outputs/apk/debug/app-debug.apk`
(relative to the repository root). Its separate `.gt6t` package ID lets it
coexist with the upstream APK. It still requires your own game files under
`/storage/emulated/0/nfsu2/` on the device. Source archives alone cannot run
the game; they require this local generation and build step.
