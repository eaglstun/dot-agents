---
name: apple-help-books
description: Apple Help Book reference for macOS apps — building a .help bundle, the hiutil search index,
  the Info.plist keys that make Help Viewer find it, and opening it from C or Objective-C. Use when adding
  native Help-menu content to a Mac app, when Help Viewer opens blank or not at all, or when deciding
  whether an app needs a bundle at all.
metadata:
  version: 1.0.0
  public: true
---

# Apple Help Books

Verified end-to-end on macOS (Darwin 25.5, Sept 2026) against a non-Xcode,
autotools-built C/C++ app (Dillo/FLTK). Everything below was observed, not
recalled.

## The one thing that decides the whole design

**A help book is resolved through the *application bundle*, not on its own.**
Help Viewer reads two keys from the app's `Info.plist`:

- `CFBundleHelpBookFolder` — the `.help` folder name inside `Contents/Resources`
- `CFBundleHelpBookName` — must match the book's `HPDBookTitle` *and* the
  `<meta name="AppleTitle">` in its `index.html`

So a bare Unix binary cannot have a help book. If the app is not already a
`.app`, adding one is part of the job, not a nice-to-have.

## Layout that works

```
Dillo.app/Contents/
  Info.plist                     CFBundleHelpBookFolder + CFBundleHelpBookName
  MacOS/dillo
  Resources/Dillo.help/Contents/
    Info.plist                   HPDBook* keys, CFBundlePackageType = BNDL
    Resources/en.lproj/
      index.html                 <meta name="AppleTitle" content="Dillo Help">
      *.html
      search.cshelpindex         built by hiutil
```

Help book `Info.plist` keys that were sufficient:

```
CFBundleIdentifier      org.example.app.help
CFBundleName            Dillo Help
CFBundlePackageType     BNDL
CFBundleSignature       hbwr
HPDBookAccessPath       index.html
HPDBookIndexPath        search.cshelpindex
HPDBookTitle            Dillo Help
HPDBookType             3
```

## hiutil: the version trap

`hiutil` 2.0 builds **CoreSpotlight** indices by default, and its own usage
string says `-f output.cshelpindex`. The `.helpindex` extension is the older
`lsm` format. Both still build:

```sh
hiutil -I corespotlight -C -a -m 3 -s en -f .../search.cshelpindex <lproj dir>
hiutil -I lsm          -C -a -m 3 -s en -f .../search.helpindex   <lproj dir>
```

`-m` (minimum term length) only accepts 1, 2 or 3. There is no `-A`
(list-anchors) mode in 2.0 — passing it fails with "problem unarchiving the
index file", which is a bad error message for "unsupported flag", not a
corrupt index.

Treat the index as a **build product**, not a committed file: a stale index is
worse than none, and `hiutil` ships with the command line tools.

## Help Viewer is now Tips.app

On current macOS there is no `Help Viewer.app` at the classic
`/System/Library/CoreServices/` path. The `help:` URL scheme and the
`com.apple.help` UTI are claimed by **`/System/Applications/Tips.app`**, which
runs under the `com.apple.helpviewer` identity. Nothing on a stock system ships
a `.help` bundle any more, so there is no local example to copy — build to spec
and verify by opening it.

Find the handler with:

```sh
/System/Library/Frameworks/CoreServices.framework/Frameworks/\
LaunchServices.framework/Support/lsregister -dump | grep -i "help:"
```

## Opening the book

From the shell (also the quickest way to test a book in isolation):

```sh
open "help:openbook=Dillo%20Help"
```

From C, with no Objective-C file required — `LSOpenCFURLRef` works and produced
no deprecation warning:

```c
#include <CoreServices/CoreServices.h>

CFBundleRef b = CFBundleGetMainBundle();
CFStringRef name = CFBundleGetValueForInfoDictionaryKey(b,
                       CFSTR("CFBundleHelpBookName"));
/* percent-escape spaces, build "help:openbook=<name>" */
LSOpenCFURLRef(cfurl, NULL);   /* noErr == 0 on success */
```

Reading `CFBundleHelpBookName` back from the running bundle doubles as the
"am I bundled with a help book?" test: absent means fall back to whatever the
unbundled build does.

## Verifying, without losing an hour

- **Read the window, don't chase it.** Help Viewer may open on a secondary
  display at negative coordinates, where `screencapture -R` and `-D` will miss
  it. Ask accessibility for its text instead:
  `osascript -e 'tell application "System Events" to tell process "Tips" to ...
  entire contents of window 1'`. To screenshot it, get `position`/`size` of
  window 1 first and pass those to `screencapture -R`.
- **`pgrep "Help"` matches every "Helper" process on the machine.** Use
  `ps -Ao comm= | grep "Tips.app"`.
- **Tips.app quits on its own** after a short idle. "Not running" a minute
  later does not mean it never opened — check
  `log show --predicate 'process == "Tips"'` for `com.apple.helpviewer:URLHandling`.
  Note `log` may be shadowed in zsh; call `/usr/bin/log`.
- **Repeated accessibility clicking into a Help menu can leave it in a damaged
  state** where it reports only the search field plus the first item and stops
  firing callbacks. That looks exactly like "macOS pruned my menu". Relaunch
  the app and enumerate a fresh process before believing it.

## AppKit's Help menu

A menu titled "Help" gets the search field automatically, and AppKit treats the
*last* menu of that name as the help menu — so keep it last in the menu array.
Custom items below the first one are kept; see the damaged-state note above
before concluding otherwise.
