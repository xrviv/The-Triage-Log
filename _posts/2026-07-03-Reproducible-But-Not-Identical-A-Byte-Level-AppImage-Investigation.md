---
title: "Reproducible, But Not Identical: A Byte-Level AppImage Investigation"
description: "A step-by-step walkthrough for diagnosing why a reproducible build's hash doesn't match — using ADAMANT Messenger v4.11.1 as a case study"
date: 2026-07-03 20:26:00 +0800
categories: [Blogging, Tech]
tags: [reproducible-builds, walletscrutiny, forensics, appimage, squashfs]
---

# Finding out exactly what differs: official vs. built ADAMANT Messenger v4.11.1 AppImage

This walkthrough uses two AppImages for the same ADAMANT Messenger v4.11.1 release:

- `official/official-ADAMANT-Messenger-4.11.1.AppImage` — downloaded directly from
  ADAMANT's GitHub release page.
- `built/built-ADAMANT-Messenger-4.11.1.AppImage` — built from source at git tag
  `v4.11.1` in a container, using `adamant_build.sh` (WalletScrutiny's verification
  script), reproducing the same steps ADAMANT's own CI (`.github/workflows/electron-linux.yml`)
  runs: `npm ci` → `npm run schema:generate` → `npm run electron:build`.

**The question this post walks through answering:** these two files do **not**
have the same SHA-256 hash. Does that mean the build is not reproducible — i.e. that
the developer could have secretly shipped something different from what the public
source code produces? Or does it mean something more mundane?

This investigation fed directly into WalletScrutiny's [published verification report
for this release](https://walletscrutiny.com/desktop/adamant.im/#verificationId=32a5aae66e7db6cf7693f92246ad595d30f1f777a80fc089aca1cd30f348d445),
and the root cause traced below was filed as [an issue against ADAMANT's own
repository](https://github.com/Adamant-im/adamant-im/issues/957).

You do not need any prior experience with AppImages, SquashFS, or binary reverse
engineering to follow this. Every command below is something you can run yourself and
see the same thing. Where domain knowledge is needed (like "what is a SquashFS
superblock"), it's explained inline before you need it.

**Tools you need**: a Linux shell, `sha256sum`, `cmp`, `diff`, `python3` (standard
library only — no pip installs needed). All standard on any Debian/Ubuntu box.

---

## Step 0 — Orient yourself

```bash
cd ~/builds/desktop/adamant/4.11.1/comparison
ls -la official/ built/
```

You should see one `.AppImage` file in each folder, both a little over 120 MB, both
named `ADAMANT-Messenger-4.11.1.AppImage` (with an `official-`/`built-` prefix so they
don't collide in the same folder if you ever move them together).

---

## Step 1 — The obvious first check: do the files match?

```bash
sha256sum official/official-ADAMANT-Messenger-4.11.1.AppImage \
           built/built-ADAMANT-Messenger-4.11.1.AppImage
```

**What you should see:** two different hashes. Also try:

```bash
ls -la official/official-ADAMANT-Messenger-4.11.1.AppImage \
       built/built-ADAMANT-Messenger-4.11.1.AppImage
```

Notice the **file sizes are close but not identical** — a difference of a few dozen
bytes out of ~120,000,000. That's your first real clue: this isn't two completely
different builds. Whatever differs is a tiny fraction of the file.

**Stop and think before continuing:** if you stopped here and only looked at the outer
file hash, you would conclude "not reproducible." Is that actually the right
conclusion? A reproducible-build check should care about *the software the AppImage
runs*, not incidental bytes in how it's packaged. The next step separates those two
things.

---

## Step 2 — Look inside, not just at the outside

An `.AppImage` file isn't really "one file" conceptually — it's a small program (the
"runtime") with a compressed filesystem image glued onto the end of it, containing the
actual application. You can ask any AppImage to unpack that filesystem for you:

```bash
cd official && chmod +x official-ADAMANT-Messenger-4.11.1.AppImage
APPIMAGE_EXTRACT_AND_RUN=1 ./official-ADAMANT-Messenger-4.11.1.AppImage --appimage-extract >/dev/null
mv squashfs-root ../official-extracted
cd ..

cd built && chmod +x built-ADAMANT-Messenger-4.11.1.AppImage
APPIMAGE_EXTRACT_AND_RUN=1 ./built-ADAMANT-Messenger-4.11.1.AppImage --appimage-extract >/dev/null
mv squashfs-root ../built-extracted
cd ..
```

Now compare the actual unpacked application trees, file by file:

```bash
diff -r official-extracted built-extracted
echo "exit code: $?"
```

**What you should see:** no output, exit code `0`. For extra confidence, hash every
file in both trees and compare the whole lists:

```bash
(cd official-extracted && find . -type f -o -type l | sort | xargs -I{} sha256sum {}) > /tmp/official_hashes.txt
(cd built-extracted    && find . -type f -o -type l | sort | xargs -I{} sha256sum {}) > /tmp/built_hashes.txt
diff /tmp/official_hashes.txt /tmp/built_hashes.txt && echo "ALL FILE HASHES IDENTICAL"
```

**Clue:** every single application file — every binary, every library, every resource
— is byte-for-byte identical. The thing people actually run (Electron + the app's JS
bundle + its native modules) is provably the same in both builds. So whatever makes the
*outer* file hash differ must live in the ~1% of the file that isn't "the app" — the
AppImage packaging wrapper itself. That's a much smaller haystack to search.

---

## Step 3 — Where, exactly, do the raw files diverge?

Now go back to comparing the two `.AppImage` files as raw bytes, but be smarter about
it than a whole-file hash. `cmp` tells you the position of the *first* byte where two
files differ:

```bash
cmp official/official-ADAMANT-Messenger-4.11.1.AppImage \
    built/built-ADAMANT-Messenger-4.11.1.AppImage
```

**What you should see:** something like
`... differ: byte 188401, line 429`. Write that number down.

That's about 188 KB into a 120 MB file — very early. (Arithmetic: `188401 bytes ÷ 1000 ≈
188 KB`, using the same decimal-KB/MB convention as "120 MB" above — 1 KB = 1000 bytes,
1 MB = 1,000,000 bytes.) On its own this single number doesn't tell you much. What you
actually want is the *shape* of the divergence: is it one small isolated difference, or
does everything change from that point on? `cmp -l` (lowercase L) lists **every**
differing byte, not just the first:

```bash
cmp -l official/official-ADAMANT-Messenger-4.11.1.AppImage \
       built/built-ADAMANT-Messenger-4.11.1.AppImage 2>/dev/null > /tmp/all_diffs.txt
wc -l /tmp/all_diffs.txt
awk 'NR==1{print "first differing byte:", $1} END{print "last differing byte:", $1}' /tmp/all_diffs.txt
```

*Why `2>/dev/null > /tmp/all_diffs.txt` and not just `> /tmp/all_diffs.txt`:* the two
files here are slightly different lengths. Once `cmp -l` walks past the end of the
shorter file, it prints one extra diagnostic line to **stderr** (something like
`cmp: EOF on built-... after byte N`) — that's not a byte-diff entry, just a notice. `2>/dev/null`
throws that notice away so only the actual `<offset> <official-byte> <built-byte>` data
lines land in the file; otherwise that stray text line would break the `awk`/`sort`
byte-counting below, since it isn't shaped like the numeric rows they expect.

*What `wc -l /tmp/all_diffs.txt` tells you:* every line `cmp -l` writes is exactly one
differing byte (`<byte offset> <official's byte value> <built's byte value>`, values in
octal). So the line count **is** the total number of bytes that differ between the two
files — e.g. a result of `106752` means 106,752 individual bytes differ, out of
~120,000,000 total (well under 0.1% of the file).

Now bucket those differing byte positions by megabyte, to see *where* in the file they
cluster:

```bash
awk '{print int($1/1000000)}' /tmp/all_diffs.txt | sort -n | uniq -c
```

**What you should see:** something like:

```
     10 0
  46469 119
  60173 120
```

*How to read this:* `awk '{print int($1/1000000)}'` takes each differing byte's offset
(column 1 of `all_diffs.txt`) and floors it to a "which megabyte is this in" bucket
number — offset 188401 becomes bucket `0` (the 1st MB, bytes 0-999999), an offset around
119,500,000 becomes bucket `119` (the 120th MB), and so on. `sort -n | uniq -c` then
counts how many lines share each bucket number and prints `<count> <bucket>` per line —
**not** `<bucket> <count>`, the count comes first. So `10 0` means "10 differing bytes
fall in bucket 0 (the very start of the file)," and `46469 119` / `60173 120` mean
"46,469 differing bytes fall in the 120th MB, and 60,173 more in the 121st (final,
partial) MB" — i.e. almost all the differing bytes are crammed into the last ~2 MB of a
120 MB file, with only a handful near the start. That lopsided count is exactly the
"two small clusters, nothing in between" shape described below.

A small number of differing bytes in the low buckets (near the start, close to byte
188401) and a much larger cluster of differing bytes in the *highest* buckets — i.e.
near the very end of the file. Nothing in the middle.

**Clue:** random corruption or a genuinely different build would scatter differences
all over the file. A concentrated cluster at the start plus a concentrated cluster at
the end is the signature of "two small, specific regions changed" — exactly what you'd
expect from packaging *metadata* (which tends to live at fixed, predictable positions),
not application *content* (which you already proved is identical in Step 2).

---

## Step 4 — What lives at that first offset?

**Plain-language definition — what's an "offset"?** A file is just a long line of
bytes, numbered starting at 0 (byte 0, byte 1, byte 2, ...). An "offset" is simply
**a position in that line, counted in bytes from the start of the file.** "Offset
188392" just means "count 188,392 bytes in from the very beginning of the file, and
stop there." It's the same idea as a page number in a book, except the unit is bytes
instead of pages. You've already been using offsets without the label: back in Step 3,
`cmp` told you the files first differ "at byte 188401" — 188401 *is* an offset.

Recall from Step 2 that an AppImage is "a small runtime program + an appended
filesystem image." The runtime itself can tell you exactly where its own appended
filesystem starts — i.e., it can tell you the offset at which that filesystem begins:

```bash
official/official-ADAMANT-Messenger-4.11.1.AppImage --appimage-offset
built/built-ADAMANT-Messenger-4.11.1.AppImage --appimage-offset
```

**What you should see:** the same number for both files — and it should look very
close to (byte 188401 minus a handful of bytes) from Step 3.

**Clue:** the ELF runtime stub — the actual executable part of the AppImage — is
identical in size in both builds (same offset both times). The divergence you found in
Step 3 begins essentially exactly where the runtime ends and the appended filesystem
image begins. So the first cluster of differing bytes isn't in the "program" part of
the AppImage at all — it's in the first few bytes of the filesystem image's own
bookkeeping header.

**Careful — a common misreading:** it's tempting to conclude "so the *real* program
begins at byte 188392, and everything before that is just filler." That's not quite
right, and it's worth confirming, not assuming. Check what's actually sitting in those
first ~188 KB:

```bash
# Confirm this really is a type-2 AppImage (the format's own magic marker)
xxd -s 8 -l 3 official/official-ADAMANT-Messenger-4.11.1.AppImage

# Is that whole leading region identical between official and built?
cmp <(head -c 188392 official/official-ADAMANT-Messenger-4.11.1.AppImage) \
    <(head -c 188392 built/built-ADAMANT-Messenger-4.11.1.AppImage) && echo "IDENTICAL"

# What's actually in there?
head -c 188392 official/official-ADAMANT-Messenger-4.11.1.AppImage | strings | grep -iE "appimage|apprun" | head -10
```

**What you should see:** the magic bytes are `41 49 02` (`AI` followed by `\x02` — the
official marker meaning "type-2 AppImage"). The first 188,392 bytes are reported
`IDENTICAL` between official and built. And the `strings` output shows things like
`extract_appimage`, `appimage_get_elf_size`, and the `--appimage-extract`/
`--appimage-mount` help text — none of which is ADAMANT's code.

**The corrected picture:** that leading region is the generic **AppImage runtime** — a
small, standard bootstrapping executable used by AppImages broadly (not written by
ADAMANT's developers, not specific to this app at all). Its only job is "when someone
runs me, find the filesystem image glued onto my own tail end, and mount or extract
it." It genuinely is a real, working program in its own right — just not *ADAMANT's*
program, which is why you just proved it's byte-identical between the two builds (there
was never any reason to expect it to differ; it's boilerplate). The application code you
actually care about — the thing you already proved is byte-for-byte identical back in
Step 2 — lives inside the filesystem image that starts at 188392, not "begins" there in
the sense of being the only real content.

---

## Step 5 — Reading the filesystem image's header

The filesystem format AppImage uses here is called **SquashFS** — a common read-only,
compressed Linux filesystem format (you've almost certainly used one before without
knowing it; many Linux live-CDs and Snap packages use it too).

**Plain-language definition — what's a "superblock"?** Think of the whole SquashFS
image (everything from offset 188392 onward) as a shipping container packed full of
files. Before you unpack the whole container, you'd want a **label on the outside**
telling you: how many items are inside, when it was packed, how it's compressed, and
where inside the container to find the "index" that lists everything. That label is
the superblock. It's not one of the *files* being stored — it's bookkeeping *about* the
filesystem itself, always sitting in the same fixed spot (the very first 96 bytes of
the SquashFS image) so any program can read it without having to unpack anything first.
"Fixed-size" here just means it's always exactly 96 bytes, arranged in the same order,
in every SquashFS image ever made — that predictability is *why* we can reliably locate
and decode it below, the same way you can reliably find page 1 of a book right after
the cover, every time.

You don't need to memorize this layout — it's public, stable (SquashFS 4.0, unchanged
since 2009), and here it is, as a table of `(field name, size in bytes, how to decode
it)`, in the order the fields appear starting right at the offset you found in Step 4:

| Field | Size | Meaning |
|---|---|---|
| `magic` | 4 bytes | Always the ASCII text `hsqs` if this really is a SquashFS image |
| `inode_count` | 4 bytes | How many filesystem objects (files/dirs/symlinks) it contains |
| `mkfs_time` | 4 bytes | **A Unix timestamp** — the moment the `mksquashfs` tool ran |
| `block_size` | 4 bytes | Compression block size |
| `fragment_entry_count` | 4 bytes | — |
| `compression_id` | 2 bytes | Which compression algorithm (1 = gzip) |
| `block_log` | 2 bytes | — |
| `flags` | 2 bytes | — |
| `id_count` | 2 bytes | — |
| `version_major` | 2 bytes | SquashFS format version (always 4) |
| `version_minor` | 2 bytes | SquashFS format version (always 0) |
| `root_inode` | 8 bytes | Pointer to the filesystem's root directory |
| `bytes_used` | 8 bytes | Total size of the actual filesystem image |
| `id_table_start` | 8 bytes | Byte offset of an internal table |
| `xattr_id_table_start` | 8 bytes | Byte offset of an internal table |
| `inode_table_start` | 8 bytes | Byte offset of an internal table |
| `directory_table_start` | 8 bytes | Byte offset of an internal table |
| `fragment_table_start` | 8 bytes | Byte offset of an internal table |
| `export_table_start` | 8 bytes | Byte offset of an internal table |

Rather than reading these by hand in a hex editor, decode them programmatically. Python's
`struct` module can unpack raw bytes according to a format string — `<I` means "a
little-endian unsigned 4-byte integer," `<Q` means "a little-endian unsigned 8-byte
integer," and so on, matching the "Size" column above.

**Before you run the full script below**, try to predict: given what Step 3 showed you
(a *small* cluster of differing bytes right at this offset, not the whole 96-byte
superblock), do you expect most of these 19 fields to match, or most to differ? Which
one field, given its meaning in the table above, seems like the single most likely
candidate to differ between two builds of the same source code run at different times?

Now check your prediction:

```bash
python3 << 'PYEOF'
import struct, datetime

FIELDS = [
    ("magic", 4, "4s"), ("inode_count", 4, "<I"), ("mkfs_time", 4, "<i"),
    ("block_size", 4, "<I"), ("fragment_entry_count", 4, "<I"),
    ("compression_id", 2, "<H"), ("block_log", 2, "<H"), ("flags", 2, "<H"),
    ("id_count", 2, "<H"), ("version_major", 2, "<H"), ("version_minor", 2, "<H"),
    ("root_inode", 8, "<Q"), ("bytes_used", 8, "<Q"), ("id_table_start", 8, "<Q"),
    ("xattr_id_table_start", 8, "<Q"), ("inode_table_start", 8, "<Q"),
    ("directory_table_start", 8, "<Q"), ("fragment_table_start", 8, "<Q"),
    ("export_table_start", 8, "<Q"),
]

# Paste the number --appimage-offset printed in Step 4 here if it's different:
OFFSET = 188392

def read_superblock(path, offset):
    vals = {}
    with open(path, "rb") as f:
        f.seek(offset)
        for name, size, fmt in FIELDS:
            vals[name] = struct.unpack(fmt, f.read(size))[0]
    return vals

sb_official = read_superblock("official/official-ADAMANT-Messenger-4.11.1.AppImage", OFFSET)
sb_built    = read_superblock("built/built-ADAMANT-Messenger-4.11.1.AppImage", OFFSET)

for name, _, _ in FIELDS:
    o, b = sb_official[name], sb_built[name]
    marker = "  <-- DIFFERS" if o != b else ""
    print(f"{name:25s} official={o!r:22} built={b!r:22}{marker}")
PYEOF
```

**What you should see:** `magic` is `b'hsqs'` for both (confirming this really is a
SquashFS image, so the table above genuinely applies). Most fields match exactly. A
small number — usually just `mkfs_time`, sometimes also `root_inode`, `bytes_used`, and
the `*_table_start` offsets — show `<-- DIFFERS`.

---

## Step 6 — Turn the raw numbers into something meaningful

`mkfs_time` is a **Unix timestamp**: the number of seconds since 1970-01-01 UTC.
Decode both values into a human-readable date:

```bash
python3 -c "
import datetime
official_ts = 1774207937  # <-- REPLACE with YOUR Step 5 'official' mkfs_time if different
built_ts    = 1783050541  # <-- REPLACE with YOUR Step 5 'built' mkfs_time if different
print('official:', datetime.datetime.fromtimestamp(official_ts, datetime.timezone.utc))
print('built:   ', datetime.datetime.fromtimestamp(built_ts, datetime.timezone.utc))
"
```

(The two numbers above are the actual values from this folder's own `official/`/`built/`
pair — running this as-is already gives you the real answer for these files. Only
replace them if you re-ran the build yourself and Step 5 printed different numbers.)

**What you should see:** the `official` timestamp lands on ADAMANT's actual v4.11.1
release date. The `built` timestamp lands on the exact minute *you* (or whoever ran
`adamant_build.sh`) started this build — check the file's own modification time with
`ls -la built/` to confirm they line up.

**This is the answer.** `mksquashfs` (the tool that packs the AppImage's filesystem
image) stamps the current wall-clock time into its own superblock, every single time it
runs, with no way to override it in this project's build process. Every build — even
of literally identical source code, run seconds apart on the same machine — gets a
different `mkfs_time`, and therefore a different outer file hash, purely because of
*when* it happened to run.

**If `root_inode`, `bytes_used`, or the `*_table_start` fields also differed for
you:** that's a real, additional, and slightly subtler effect worth understanding, not
a mistake. `mksquashfs` occasionally encodes small pieces of data (like inode metadata)
using a variable-length scheme, where a build's specific timestamp can, by coincidence,
need one extra byte to represent versus another build's timestamp. If that happens, one
byte gets inserted before the internal tables, and every table-start offset after that
point shifts by exactly that many bytes — which is exactly why you'd see several
offsets differ by the same small amount (e.g. all off by +1) rather than by unrelated
random amounts. It's a byte-length side-effect of the same root cause (the timestamp),
not a second, independent, unexplained difference. If you see this, verify it yourself:
subtract the official's `bytes_used` from the built's `bytes_used` — that delta should
match how much every other later offset shifted by.

---

## Step 7 — What about the *second* cluster of differing bytes, near the end?

Step 3 showed differences clustered at two places: right after this superblock, and
also near the very end of the file. `bytes_used` (which you just decoded) tells you
exactly where SquashFS considers its own filesystem image to end. Compare that to the
actual file size:

```bash
python3 -c "
import os
OFFSET = 188392
BYTES_USED = 119743657  # <-- REPLACE with YOUR Step 5 'official' bytes_used if different
official_size = os.path.getsize('official/official-ADAMANT-Messenger-4.11.1.AppImage')
built_size = os.path.getsize('built/built-ADAMANT-Messenger-4.11.1.AppImage')
print('bytes after the SquashFS image, official:', official_size - OFFSET - BYTES_USED)
print('official file size:', official_size, ' built file size:', built_size, ' delta:', official_size - built_size)
"
```

**What you should see:** the SquashFS filesystem image itself does *not* extend all the
way to the end of the `.AppImage` file — there's roughly another ~128 KB tacked on
after it. That's `appimagetool` (the tool that assembles the final `.AppImage` from the
runtime + the SquashFS image) appending its own trailer — commonly an embedded digest
of the whole image, used for AppImage's optional update/verification tooling.

**Clue, and the reason this matters:** that trailer is computed **over** the SquashFS
image it's attached to — the same SquashFS image whose superblock timestamp you just
proved differs between builds. So the trailer is a *downstream consequence* of Step 6's
finding, not a second, separate, unexplained source of difference. Confirm this
yourself by peeking at the trailer bytes:

```bash
python3 -c "
OFFSET = 188392
# Each file gets its OWN bytes_used from Step 5 -- not the same number reused for both.
# official and built can (and here, do) have slightly different bytes_used, so using
# only one value for both files would measure the built trailer starting from the
# wrong spot.
BYTES_USED = {
    'official': 119743657,  # <-- REPLACE with YOUR Step 5 'official' bytes_used if different
    'built':    119743658,  # <-- REPLACE with YOUR Step 5 'built' bytes_used if different
}
for label, path in [('official', 'official/official-ADAMANT-Messenger-4.11.1.AppImage'),
                     ('built', 'built/built-ADAMANT-Messenger-4.11.1.AppImage')]:
    trailer_off = OFFSET + BYTES_USED[label]
    with open(path, 'rb') as f:
        f.seek(0, 2); size = f.tell()
        f.seek(trailer_off)
        trailer = f.read(size - trailer_off)
        print(label, 'trailer length:', len(trailer), 'first 16 bytes:', trailer[:16])
"
```

You'll see the trailer starts with zero-padding (block alignment) and then a run of
what looks like random/compressed-looking bytes — consistent with a checksum or digest,
not readable content. Its length differs between the two builds too (you may have
noticed the two `.AppImage` files aren't exactly the same size) — again, a downstream
effect of the timestamp-dependent content it's summarizing, not evidence of anything
else changing.

**Putting the whole 26-byte size difference to rest, in plain language:** the official
and built files aren't the same total size (a 26-byte difference in this dataset). That
26 isn't one single change — it's two separate small changes that happen to add up:

1. The SquashFS *data* region (`bytes_used`) is **1 byte longer** in the built file
   than in the official file (`119743658` vs `119743657`). This is the same kind of
   1-byte shift you may have seen in Step 5/6, on the `id_table_start`,
   `directory_table_start`, etc. fields — a small, mechanical side-effect of the
   timestamp being encoded slightly differently.
2. The trailer *after* that data region is **27 bytes shorter** in the built file
   (`128407` official vs `128380` built — check this yourself with the script above).
   The trailer is a checksum/digest computed over the data region, so a different data
   region (because of the different timestamp) naturally produces a different, and here
   slightly shorter, digest.

Put those two together: the built file's data region eats up 1 extra byte, but its
trailer gives back 27, so the built file ends up `27 - 1 = 26` bytes *smaller* overall
than the official file — exactly matching the file-size delta you saw back in Step 1.
Nothing mysterious is unaccounted for: every one of those 26 bytes is explained by the
same root cause you already found in Step 6 (the build timestamp), just showing up here
as a size difference instead of a content difference.

---

## Putting it together

By this point you've independently proven, in order:

1. The application content — every file inside the AppImage — is byte-for-byte
   identical between the official release and a from-source rebuild (Step 2).
2. The only differences between the raw files live in two small, fixed-position
   regions: the SquashFS superblock header (Step 3/4) and a trailer after it (Step 7).
3. The superblock's `mkfs_time` field — a build-time Unix timestamp with no
   `SOURCE_DATE_EPOCH`-style override in this project's build tooling — is the root
   cause (Step 5/6), and any other superblock-field or trailer differences are
   mechanical, explainable side-effects of that one timestamp (Step 6/7), not separate
   unexplained changes.

**Conclusion:** this is a packaging-tool artifact, not evidence that the published
binary contains different code than the public source. The application is
reproducible; the *packaging step* embeds a timestamp that isn't. This is the same
category of difference [WalletScrutiny](https://walletscrutiny.com/) classifies across
several packaging formats (AppImage, tar.xz, .deb, .rpm, and Flatpak all have their own
version of this same problem) — a distinction between "the shipped software matches the
public source" (what actually matters) and "the outer container file is byte-identical"
(a much stricter, and often unrelated, bar).

## Quick reference — every command in one place

```bash
# whole-file hash + size
sha256sum official/*.AppImage built/*.AppImage
ls -la official/*.AppImage built/*.AppImage

# extract and compare the actual application content
APPIMAGE_EXTRACT_AND_RUN=1 official/official-ADAMANT-Messenger-4.11.1.AppImage --appimage-extract
APPIMAGE_EXTRACT_AND_RUN=1 built/built-ADAMANT-Messenger-4.11.1.AppImage --appimage-extract
diff -r squashfs-root <other-extracted-tree>   # do this for both, then diff the two trees

# find where the raw files diverge
cmp official/*.AppImage built/*.AppImage
cmp -l official/*.AppImage built/*.AppImage | awk '{print int($1/1000000)}' | sort -n | uniq -c

# find where the SquashFS image begins
official/*.AppImage --appimage-offset
built/*.AppImage --appimage-offset

# decode and compare the SquashFS superblock (see Step 5 for the full script)
python3 <superblock-decoder-script>
```
