# `window.Native`

Available inside any app imported into the Web2App container. Standard web APIs
are unaffected and should still be your first choice — `getUserMedia`,
`geolocation`, `IndexedDB`, `MediaRecorder`, `navigator.share` and the rest all
work, because each app runs on a real https origin.

`Native` only covers what the web platform cannot do on Android.

## Detecting it

```js
if (window.Native) {
  // running inside the container
} else {
  // running in a normal browser — fall back to web APIs
}
```

The object exists before your own scripts run. If you need certainty on older
WebViews, listen once for the `nativeready` event on `window`.

## Instant actions

```js
Native.toast('Saved');                 // brief Android toast
Native.vibrate(30);                    // milliseconds, capped at 5000
Native.keepAwake(true);                // stop the screen sleeping
Native.setStatusBarColor('#1F7A5C');   // recolour the bar above your app
Native.share('Text to send', 'Title'); // the real Android share sheet
Native.close();                        // quit back to the home screen
```

## Storage that survives a cache wipe

`localStorage` and IndexedDB can be cleared by Android under storage pressure.
These files live in the app's own directory and only go when the user deletes the
app.

```js
Native.files.write('notes.txt', 'hello');       // -> true / false
Native.files.read('notes.txt');                 // -> string, or null
Native.files.remove('notes.txt');               // -> true / false
Native.files.list();                            // -> [{name, size, modified}]

Native.files.writeJSON('save.json', { level: 7 });
const save = Native.files.readJSON('save.json', { level: 1 });
```

Names may include subfolders (`saves/slot1.json`). Path traversal is rejected.

## The phone's real Downloads folder

```js
// plain text
Native.download('export.csv', 'a,b,c\n1,2,3');

// binary, as a data URL
const url = canvas.toDataURL('image/png');
Native.download('drawing.png', url);
```

Returns the saved location as a string, or `null` if it failed. Ordinary
`<a download>` links and blob downloads are also intercepted and saved for you,
so existing code does not need changing.

## Picking a file

```js
const file = await Native.pickFile('image/*');   // or '*/*'
if (file) {
  console.log(file.name, file.type, file.size);
  img.src = file.dataUrl;
}
```

Resolves to `null` if the user backed out. Files above 32 MB are refused.

## Notifications

```js
const result = await Native.notify('Export finished', '412 rows written');
// { ok: true } or { ok: false, reason: 'denied' }
```

The first call triggers Android's permission prompt on Android 13 and newer.
Tapping the notification reopens the app.

## Device information

```js
const info = Native.info();
// { appName, appId, platform: 'android', androidVersion, sdkInt,
//   model, manufacturer, container: 'Web2App', appBytes, freeBytes }
```

## Writing apps that work in both places

```js
const save = (name, data) =>
  window.Native
    ? Native.files.writeJSON(name, data)
    : localStorage.setItem(name, JSON.stringify(data));

const toast = (msg) =>
  window.Native ? Native.toast(msg) : console.log(msg);
```

That pattern lets the same file run in Chrome while you build it and gain the
native abilities the moment it is imported.
