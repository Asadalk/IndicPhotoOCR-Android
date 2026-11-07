# IndicPhotoOCR 2.0: Android app source (version 2.0.0-alpha7.5)

An offline Android app that reads text in photos in English and 11 Indian languages (Hindi, Bengali, Marathi, Tamil, Telugu, Kannada, Malayalam, Gujarati, Punjabi, Odia, Assamese). Everything runs on the phone with ONNX Runtime; the app has no internet permission.

This zip has everything needed to change the code and rebuild the app on another machine: the source, the 14 ONNX models, the build scripts, the Play Store graphics and the docs. The APK/AAB are not included (~465 / ~335 MB); build them with the steps below.

---

## 1. Quick start (Linux x86_64)

```bash
unzip IndicPhotoOCR2-app-source-7.5.zip && cd IndicPhotoOCR2-app
bash setup_toolchain.sh     # once: JDK 17 + Android SDK 36 + bundletool into ./toolchain (~1.5 GB, needs internet)
bash build_apk.sh           # Gradle 8.13 is downloaded by the wrapper on first run
```

Outputs:
- `IndicPhotoOCR2.apk`: one installable APK for testing. Copy it to the phone and install it (allow "install unknown apps").
- `android/app/build/outputs/bundle/release/app-release.aab`: the app bundle for Google Play.

**Important:** the models are in a separate install-time asset pack (`android/model_pack`, needed for Google Play's 200 MB limit). Because of that, a plain `./gradlew assembleRelease` APK does **not** contain the models, and the app will say "Could not load the models". Always make test APKs with `build_apk.sh`, which runs bundletool `--mode=universal`.

The first build downloads Gradle and the Maven dependencies, so it needs internet and takes ~5–10 min. Later builds take ~1–2 min. Building needs ~4 GB RAM (`android/gradle.properties` sets `-Xmx4g`).

### Android Studio (Windows / macOS / Linux)
Open the `android/` folder in Android Studio (Narwhal 2025.1 or newer, for Android Gradle Plugin 8.11). Let it sync. Studio will create `local.properties` with your SDK path itself. Then use **Build → Generate App Bundles or APKs → Generate Bundles**, or press Run with a phone connected over USB (developer options → USB debugging); Run deploys the asset pack too. To make a single APK from the bundle, use bundletool as in `build_apk.sh`. Use JDK 17 (Settings → Build Tools → Gradle → Gradle JDK).

### Installing straight to a connected phone
```bash
toolchain/sdk/platform-tools/adb install -r IndicPhotoOCR2.apk
```

---

## 2. What is in the zip

```
IndicPhotoOCR2-app/
├── README.md                  this file
├── setup_toolchain.sh         installs JDK 17 + Android SDK locally (Linux)
├── build_apk.sh               builds the .aab (for Play) and one installable test APK
├── android/                   the Gradle project (open this in Android Studio)
│   ├── settings.gradle.kts    modules :app, :ocrcore, :verify, :model_pack
│   ├── build.gradle.kts       plugin versions (AGP 8.11.2, Kotlin 2.0.21)
│   ├── gradlew, gradle/       Gradle 8.13 wrapper
│   ├── model_pack/            install-time asset pack: src/main/assets/models/ = the 14 ONNX models + manifest.json
│   ├── app/                   the Android app (UI)
│   │   ├── build.gradle.kts   versionCode/versionName, minSdk 26, targetSdk 36, arm64 only, ONNX Runtime 1.22.0
│   │   └── src/main/
│   │       ├── kotlin/com/iitj/indicphotoocr/app/
│   │       │   ├── MainActivity.kt   camera/gallery, progress, drawing boxes, result card, side menu
│   │       │   ├── InfoActivity.kt   About Us / Terms of Service / Privacy Policy / Open-source licences screens
│   │       │   ├── ModelStore.kt     loads models straight from the APK; saves the accelerator choice
│   │       │   └── ReadingOrder.kt   puts recognised words into lines (top-to-bottom, left-to-right)
│   │       ├── res/                  layouts, strings (all app text), colours, icons, logos
│   │       ├── assets/licenses/      NOTICE.txt + full licence texts shown in "Open-source licences"
│   │       └── AndroidManifest.xml
│   ├── ocrcore/               pure-Kotlin OCR engine shared by the app and :verify
│   │   └── src/main/kotlin/com/iitj/indicphotoocr/core/
│   │       ├── OcrEngine.kt   detection → script ID → reading → overlap pruning; accelerator tuning
│   │       └── RgbImage.kt    image resize/crop/normalise helpers (match the Python reference exactly)
│   └── verify/                runs the same engine on a desktop/server JVM to compare with Python
├── tools/                     Python helpers used to make the models and check the app (section 6)
├── store/                     Play Store icon (512×512) and feature graphic (1024×500)
└── docs/
    ├── PLAY_STORE_GUIDE.md    how to publish on Google Play, costs, checklist
    ├── LICENCE_AUDIT.md       licences of every model/library, the open issues, and what to do
    └── CHANGELOG.md           what changed in each test build
```

---

## 3. How the app works

1. **Photo in.** From the camera (system camera app), the gallery, or "Share" from another app. It is decoded with EXIF rotation applied, long side ≤ 4096 px, and shown at once.
2. **Detection.** `yolo.onnx` (YOLO11n trained on BSTD + MIST at 1920 px) runs on a letterboxed 1920-px image. Boxes with conf ≥ 0.25 are kept, then NMS at IoU 0.7, ≤ 300 boxes. The photo is redrawn with white boxes.
3. **Script ID.** `script_id.onnx` (EfficientNet-B0, 224 px, distilled from our EfficientNetV2-L) assigns each box one of 12 scripts. This is used only to pick the reader; nothing is shown to the user (7.3).
4. **Reading.** Each word crop is read by the PARSeq model of its script (`parseq_<lang>.onnx`, int8).
5. **Overlap pruning.** Duplicate, fragment and phrase boxes of the same text are removed: base rule + D1 (cross-script pairs) + D2 (per-script confidence percentile, table in `manifest.json → "prune"`). D3 (re-reading jittered crops) is in the code but off, because it is too slow on phones. Select it with `-PpruneMode=d123`; the default is `d12`.
6. **Result.** The final boxes are drawn and the words are put into reading order. The text card has Copy and Share buttons.

**Loading:** the models are memory-mapped straight from the installed asset-pack APK (stored uncompressed: `noCompress "onnx"`; if a file ever arrives compressed, `ModelStore` reads it into memory instead). Hindi and English load and warm up when the app opens. The other 10 readers preload in the background on phones with ≥ 5.5 GB RAM; otherwise they load on first use, with at most 3 kept.

**Accelerators:** on first launch after an install/update, `OcrEngine.tuneAccelerators` tries NNAPI (GPU/NPU, fp32 and fp16) and XNNPACK for detection and script ID on a synthetic test image. One is kept only if it gives the same boxes/scripts as the CPU and is ≥ 10 % faster. The readers always run on the CPU. About Us shows what is in use.

---

## 4. Models (`android/model_pack/src/main/assets/models/`)

| File | What | Size |
|---|---|---|
| `yolo.onnx` | text detector, YOLO11n BSTD+MIST 1920, fp32 | ~11 MB |
| `script_id.onnx` | 12-class script identifier, EfficientNet-B0, fp32 | ~16 MB |
| `parseq_<lang>.onnx` × 12 | PARSeq readers, int8 (MatMul only; the patch-embed Conv stays fp32) | ~33 MB each |
| `manifest.json` | class list, charsets, input sizes, normalisation, detector thresholds, pruning table | — |

To replace a model, put the new ONNX file under the same name and update `manifest.json` if its classes, charset or input size changed. The tools in section 6 show how the current files were made (they use the paths of the original server, so edit the paths at the top).

Accuracy of this exact pipeline (BSTD, official metric, average of 10 languages, P / R / F1 / WRR / CRR):
- validation (526 images): 59.26 / 70.80 / 62.33 / 45.94 / 57.69
- test (1,319 images, one run): 53.28 / 57.30 / 53.57 / 40.93 / 57.91

---

## 5. Common changes

| Want to… | Edit |
|---|---|
| Change any text in the app (tagline, About, Terms, Privacy) | `app/src/main/res/values/strings.xml` |
| Change the source-code link or licence list | `strings.xml → source_url`, `app/src/main/assets/licenses/NOTICE.txt` |
| Change colours | `app/src/main/res/values/colors.xml` |
| Change the screen layout | `app/src/main/res/layout/activity_main.xml`, `activity_info.xml`, `drawer_header.xml`, `menu/drawer.xml` |
| Change how boxes are drawn | `MainActivity.kt → drawPlainBoxes` |
| Change the icon | `python tools/make_icon.py` (needs `pip install fonttools pillow` and the Lohit Devanagari + DejaVu fonts), or replace `res/drawable/ic_launcher_foreground.xml` |
| Change the app logo / credit line | `res/drawable-nodpi/logo_indicphotoocr.png`; credit text = `strings.xml → credit_line, supported_by` (no institute/Bhashini logos, by decision) |
| Change detection threshold / NMS | `model_pack/src/main/assets/models/manifest.json → yolo.conf / yolo.iou` |
| Change pruning | `OcrEngine.kt → prune / pruneD12 / pruneD3`; keep `tools/dedup_boxes.py` in sync |
| App ID (permanent on Play) | `app/build.gradle.kts → applicationId` = `io.github.asadalk.indicphotoocr`; do not change after the first upload |
| **Make a new version** | `app/build.gradle.kts`: raise `versionCode` by 1 and set `versionName` |
| App name on the home screen | `app/build.gradle.kts → manifestPlaceholders["appLabel"]` |

---

## 6. Python tools (`tools/`)

These ran on the lab server (conda env `indicphotoocr`). They need the IndicPhotoOCR repo and the BSTD dataset, which are not in this zip.

| Script | Purpose |
|---|---|
| `export_onnx.py` | exports YOLO, the script identifier and the PARSeq readers to ONNX |
| `compress_models.py` | makes the fp16/int8 versions |
| `make_bundle.py`, `make_bundle_all.py` | assembles the models folder (canonical names + manifest); copy the result to `android/model_pack/src/main/assets/models/` |
| `reference_pipeline.py`, `reference_pipeline_all.py` | Python copy of the app pipeline; scores BSTD val with the official metric |
| `dedup_boxes.py` | Python copy of the app's overlap pruning (base rule) |
| `make_icon.py` | generates the app icon and the Play Store graphics |

### Checking the Kotlin engine against Python (`:verify`)
```bash
cd android && ./gradlew :verify:installDist
PRUNE_MODE=d12 verify/build/install/verify/bin/verify ../path/to/models - <folder of .png images> out.json 8
```
This runs the exact app engine on a computer, so its output can be compared word by word with `reference_pipeline_all.py`.

---

## 7. Signing and release

The build is signed with the **debug key** (`signingConfig = debug` in `app/build.gradle.kts`). That is fine for test installs, but **not** for Google Play, and a debug-signed APK cannot update an app signed with another machine's debug key: uninstall the old app first. For Play: create an upload key, add a `signingConfigs.release` block that uses it, and upload `app-release.aab`. Already done for Play: .aab output, models in an install-time asset pack (base ~14 MB, model pack ~335 MB download), targetSdk 36, 16 KB-aligned native libraries. See `docs/PLAY_STORE_GUIDE.md`.

---

## 8. Troubleshooting

| Problem | Fix |
|---|---|
| `SDK location not found` | run `setup_toolchain.sh`, or create `android/local.properties` with `sdk.dir=/path/to/Android/Sdk` |
| `Unsupported class file major version` / Kotlin JVM target errors | use **JDK 17** (`JAVA_HOME`) |
| Gradle out of memory | raise `-Xmx` in `android/gradle.properties` |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` on the phone | an older build with another signing key is installed: uninstall it first |
| `INSTALL_FAILED_NO_MATCHING_ABIS` | the APK is arm64 only; emulators need `x86_64` added to `abiFilters` (+ ~40 MB) |
| App says "Could not load the models" | the APK was built with `assembleRelease` (no asset pack): use `build_apk.sh`; or the `model_pack/src/main/assets/models` files are missing |
| Phone with < 4 GB RAM is slow or closes | expected: the models need ~1–1.5 GB; the readers are loaded lazily on such phones |

Source: https://github.com/Asadalk/IndicPhotoOCR-Android (AGPL-3.0). VL2G, IIT Jodhpur.
