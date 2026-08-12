# Web2App — container

One Android app. Install it once. After that, any HTML file becomes a real app on
the home screen in about ten seconds, entirely on the phone, with no build tools,
no hosting and no internet.

This is Milestone 1. Milestone 2 (a standalone `.apk` per app) reuses this exact
binary as its template, which is why it comes first.

---

## What the user experiences

1. Open Web2App, tap **Import an app**, pick `my-app.html` or `my-app.zip`.
2. It says the app is ready and offers to add an icon.
3. The icon lands on the home screen with their app's name and artwork.
4. Tapping it opens their app fullscreen. No address bar, no browser, its own
   card in the recent-apps switcher.

Nothing is uploaded. Nothing is compiled. It works in aeroplane mode.

---

## Why this is not "a browser tab"

Every imported app is served over **`https://<id>.web2app.local/`** from the
phone's own storage. That one decision is doing a lot of work:

- **`https` means a secure context.** A WebView pointed at `file://` silently
  disables `getUserMedia`, geolocation, service workers and `crypto.subtle`.
  Served this way, all of them work.
- **A unique host per app means a unique origin.** Two imported apps can both use
  `localStorage.setItem('save', ...)` and never see each other's data.
- **Nothing is fetched over the network**, so it is as fast as reading a file,
  because it is reading a file.

On top of that, the container hands each app a `window.Native` object with the
things the web platform genuinely cannot do on Android: writing to the phone's
real Downloads folder, storage that outlives a cache wipe, Android notifications,
the system share sheet, the vibrator, and a keep-awake lock. See
[NATIVE-API.md](NATIVE-API.md).

---

## Build it once

You need a free GitHub account. You do not need Android Studio, Gradle, a JDK,
or any experience. The compile happens on GitHub's machines.

1. Create a new repository (public or private, either works).
2. Upload every file and folder from this project to it, including the hidden
   `.github` folder. From a laptop, drag them into the "Add file → Upload files"
   box. From a phone, unzip first and upload in two passes: the `app` folder and
   the root files, then create `.github/workflows/build-apk.yml` by hand with
   "Add file → Create new file" and paste the contents in.
3. Open the **Actions** tab, enable workflows if asked, choose
   **Build Web2App APK**, then **Run workflow**.
4. Wait about eight minutes. Open the finished run and download the
   **Web2App-apk** artifact from the bottom of the page.
5. Unzip it on the phone and open the `.apk` from the Files app. Android asks
   permission to install from Files the first time — that is normal, and one-off.

### Making updates install cleanly

By default each build generates a throwaway signing key, so build 2 will not
install over build 1 (Android treats a different key as a different app). To fix
that, generate one key and keep it:

```
keytool -genkeypair -v -keystore release.jks -alias web2app \
  -keyalg RSA -keysize 2048 -validity 10000
base64 -w0 release.jks > release.jks.b64
```

Then in the repo: **Settings → Secrets and variables → Actions**, add
`KEYSTORE_BASE64` (the contents of `release.jks.b64`), `KEYSTORE_PASSWORD`,
`KEY_PASSWORD` and `KEY_ALIAS` (`web2app`). Keep `release.jks` somewhere safe —
losing it means every user has to uninstall and reinstall.

---

## What is in here

```
app/src/main/java/com/web2app/container/
  ManagerActivity.java   the container's own screen: import, pin, rename, delete
  AppActivity.java       one imported app running in its own window
  AssetServer.java       serves each app over https from its own origin
  Importer.java          reads .html or .zip into private storage
  AppStore.java          the app list and where files live
  IconFactory.java       home screen artwork, shipped or generated
  Shortcuts.java         pinning an icon to the launcher
  NativeBridge.java      the capabilities a browser tab does not get
.github/workflows/build-apk.yml   the cloud build
```

Roughly 1,900 lines. No third-party dependencies beyond AndroidX.

---

## Known limits, stated plainly

- **Background execution is still impossible.** Android suspends WebView content
  when it leaves the screen. If an app needs to record or sync while the phone is
  in someone's pocket, that needs a foreground service written in Java — a real
  extension point on this codebase, but not something the container does today.
- **All imported apps share one entry in Settings → Apps.** They have separate
  home screen icons, separate recents cards and separate storage, but Android
  sees one installed package. Milestone 2 removes this.
- **Uninstalling the container removes every imported app.** Their files live in
  its private storage.
- **The first build's key is throwaway** unless you add the secrets above.
- **`minSdk` is 26** (Android 8, 2017). A Galaxy S24 FE is far past that.

## Security decisions worth knowing about

- Zip extraction rejects any entry whose resolved path escapes the target
  directory. Without that check, importing a crafted zip could overwrite files
  elsewhere in the app's storage.
- `file://` access is switched off in the WebView entirely; local files are only
  reachable through the https asset server.
- Any navigation away from an app's own origin is handed to the real browser, so
  the native bridge is never exposed to a page that was not imported.
- `NativeBridge` file methods reject `..`, absolute paths and backslashes, and
  re-check the canonical path before touching anything.
