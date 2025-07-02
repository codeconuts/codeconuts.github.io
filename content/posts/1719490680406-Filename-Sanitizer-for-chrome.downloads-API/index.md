---
title: "Filename Sanitizer for chrome.downloads API"
date: 2024-06-27
draft: false
description: "a description"
tags: ["coding", "chrome extension"]
---

# Filename Restrictions and Sanitizing Regexs

If the filename contains some invalid Unicode characters, or matches a reserved keyword in NTFS, then `chrome.downloads.download` will throw `Error: Invalid filename`.

1. **At any position** (start/middle/end), the following Unicode characters are invalid:

   - Control characters `\p{Cc}` (`\u{0}` is allowed at the middle)
   - `:` `?` `"` `*` `<` `>` `|` `~` (NTFS reserved characters)
   - `\` and `/` (NTFS & Chrome treat them as path separators instead of a character in filename, so Invalid filename error will NOT occur)
   - Format characters `\p{Cf}`
   - Non-characters `\p{Cn}`

   Zero-width joiner `\u{200D}` is commonly used to composite emojis and form a new emoji. However, this character is invalid as well, as its category is "Format characters".

   The regex is `/[:?"*<>|~/\\\u{1}-\u{1f}\u{7f}\u{80}-\u{9f}\p{Cf}\p{Cn}]/gu`

2. **At the start/end of filename**, the following Unicode characters are invalid:

   - NUL character `\u{0}`
   - Line separator `\p{Zl}`
   - Paragraph separator `\p{Zp}`
   - Space separators `\p{Zs}`
   - A dot `.` (rule by NTFS)

   The regex is `/^[.\u{0}\p{Zl}\p{Zp}\p{Zs}]|[.\u{0}\p{Zl}\p{Zp}\p{Zs}]$/gu`

3. **Reserved keywords in NTFS**: Filenames of `CON`, `PRN`, `AUX`, `NUL`, `COM1`, `COM2`, `COM3`, `COM4`, `COM5`, `COM6`, `COM7`, `COM8`, `COM9`, `LPT1`, `LPT2`, `LPT3`, `LPT4`, `LPT5`, `LPT6`, `LPT7`, `LPT8`, `LPT9` (case-insensitive), with or without file extension, are all invalid.

   The regex is `/^(CON|PRN|AUX|NUL|COM[1-9]|LPT[1-9])(?=\.|$)/gui`

Other categories of Unicode characters, including private-use `\p{Co}` and surrogate pairs `\p{Cs}`, are allowed at any position.

## Final regex

The final regex (for JavaScript) replacing all invalid characters with `_`:

```js
var pattern = /[:?"*<>|~/\\\u{1}-\u{1f}\u{7f}\u{80}-\u{9f}\p{Cf}\p{Cn}]|^[.\u{0}\p{Zl}\p{Zp}\p{Zs}]|[.\u{0}\p{Zl}\p{Zp}\p{Zs}]$|^(CON|PRN|AUX|NUL|COM[1-9]|LPT[1-9])(?=\.|$)/gui;
filename.replaceAll(pattern, '_');
```

*Note: These characters/filenames are also invalid in NTFS, but sometimes NTFS just deletes the character instead of displaying an error.*

----------

# Determining the Invalid Unicode characters

I wrote a script to try each Unicode character, and ran it on Google Chrome 126.0.6478.116 on Windows 11 23H2 22631.3737.

What the script does:

1. Call `serviceWorker.postMessage('')` in SW console to start/stop the for loop
2. Create an empty blob and an Object URL to the blob in offscreen document
3. Call `chrome.downloads.download`, pass the Unicode character as filename and the URL created
4. Try to catch `Error: Invalid filename`. The character is invalid if error caught, otherwise valid.
5. Store the result in `chrome.storage.local`
6. Clear the download history periodically to maintain performance

The invalid Unicode characters at the start/end are found. Within the set, some characters are invalid at the middle of a filename as well. So we run the script again, replacing `filename: char` with

```js
filename: `0${char}0`
```

**background.js**:

```js
var creating;

async function setup_offscreen_document() {
    const offscreenUrl = chrome.runtime.getURL("offscreen.html");
    const existingContexts = await chrome.runtime.getContexts({
        contextTypes: ['OFFSCREEN_DOCUMENT'],
        documentUrls: [offscreenUrl]
    });

    if (existingContexts.length > 0) {
        return;
    }

    if (creating) {
        await creating;
    } else {
        creating = chrome.offscreen.createDocument({
            url: "offscreen.html",
            reasons: ['BLOBS'],
            justification: 'Create object URLs',
        });
        await creating;
        creating = null;
    }
}

var stop = true;
var url;

async function test(code) {
    var char = String.fromCodePoint(code);
    try {
        chrome.downloads.cancel(await chrome.downloads.download({ url: url, filename: char }));
        chrome.storage.local.set({ [code]: { char: char, invalid: false } });
    } catch (error) {
        if (error.message == "Invalid filename")
            chrome.storage.local.set({ [code]: { char: char, invalid: true } });
        else
            throw error;
    }
}

async function loop() {
    await setup_offscreen_document();
    url = await chrome.runtime.sendMessage(undefined, {});
    var start = (await chrome.storage.local.get('last'))['last'];
    if (start == undefined)
        start = 0;
    for (var i = start + 1; i <= 0x10ffff; i++) {
        await test(i);
        await chrome.storage.local.set({ last: i });
        if ((i - start) % 10000 == 0)
            chrome.browsingData.removeDownloads({});
        if (stop)
            break;
    }
}

function start_loop() {
    stop = false;
    loop();
}

function stop_loop() {
    stop = true;
}

self.addEventListener('message', function (e) {
    if (stop)
        start_loop();
    else
        stop_loop();
});
```

**manifest.json**:

```json
{
    "manifest_version": 3,
    "name": "Unicode Test",
    "version": "1.0",
    "description": "Test.",
    "background": {
        "service_worker": "background.js",
        "type": "module"
    },
    "permissions": [
        "downloads",
        "storage",
        "unlimitedStorage",
        "offscreen",
        "browsingData"
    ]
}
```