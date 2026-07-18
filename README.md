# newtua-unrar

A forced fork of [`unrar`](https://github.com/muja/unrar.rs) — the Rust
wrapper around `libunrar` for reading and extracting RAR archives. Used by
[newtua](https://github.com/new-the-unarchiver) as a plain dependency under
the library name `unrar`, so nothing changes for code that consumes it.

## This is a forced fork

It exists to unblock our own build, not as a product of its own. We do **not
develop it**, do **not accept outside changes**, and make **no maintenance
promises**.

**Upstream:** [muja/unrar.rs](https://github.com/muja/unrar.rs), version 0.5.8, licence `MIT OR Apache-2.0`.

**Why the fork exists:** libunrar's callback can hand us dangerous pointers —
a null buffer for `UCM_PROCESSDATA`, or a null/misaligned volume-name pointer
for `UCM_CHANGEVOLUMEW` that, if copied naively for a fixed number of
characters, also over-reads past the buffer at volume boundaries — and
without guarding against them, extracting multi-volume archives aborts the
process with SIGABRT.

We will drop this fork and go back to the upstream crate as soon as upstream
meets our needs. If you want this code for its own sake, take the upstream
crate, not our fork.

The fix has been submitted upstream: [muja/unrar.rs#77](https://github.com/muja/unrar.rs/pull/77).
**Note:** as submitted, that PR only guards against a null/misaligned
pointer — the over-read guard was added later on our side and hasn't been
folded into the PR yet. The fork goes away once upstream accepts the full fix
and cuts a release with it.

## What's actually fixed

**Target crate:** `unrar` (muja/unrar.rs), version 0.5.8.

### The problem

Handling a **multi-volume RAR** archive can abort the whole process with
`SIGABRT` at a volume boundary. We saw two distinct failures, both inside
`Internal::callback`:

1. **Extraction** reliably crashes with `SIGABRT` when a file's data crosses
   a volume boundary (a null pointer in `UCM_PROCESSDATA`).
2. **Listing/extraction** *intermittently* crashes with `SIGABRT` on volume
   changes, caused by how the original code reads the wide string naming the
   next volume in `UCM_CHANGEVOLUMEW`.

### Root cause

In `Internal::callback` (`src/open_archive.rs`), the original (unpatched)
code for `UCM_CHANGEVOLUMEW` looked like this:

```rust
let next =
    unsafe { widestring::WideCString::from_ptr_truncate(p1 as *const _, 2048) };
user_data.1 = Some(next);
```

There are two independent sources of undefined behavior here, and both need
guarding against:

1. **Null/misaligned pointer.** `from_ptr_truncate` copies via
   `ptr::copy_nonoverlapping`, whose safety contract requires the source to
   be **non-null** and **aligned** to `widestring::WideChar` (`u32` on Unix,
   `u16` on Windows). At a volume change, libunrar can pass exactly such a
   pointer.
2. **Over-read.** Even when the pointer is non-null and aligned,
   `from_ptr_truncate(p, 2048)` first **unconditionally copies all 2048
   elements** via `ptr::copy_nonoverlapping`, and only *then* truncates the
   result at the nul terminator. The real volume-name buffer is almost
   always shorter than 2048 characters, so this call reads up to 8 KiB past
   its end — and it does so even when the pointer itself is perfectly valid.
   A null/alignment check alone doesn't cover this hazard.

For `UCM_PROCESSDATA`: `std::ptr::slice_from_raw_parts(p1, p2)` +
`&*raw_slice` gets built even when `p1` is null or `p2 <= 0`. Building a Rust
reference from a null pointer is undefined behavior **even for a zero-length
slice** (`p1` here is `*const u8`, alignment 1, so only null/size matter).

The resulting UB gets caught by Rust's runtime pointer checks (debug
assertions / nightly) and turns into a panic that can't unwind across the C++
FFI boundary, so it becomes `SIGABRT`. The `UCM_CHANGEVOLUMEW` case depends on
timing/memory layout, which is why it only shows up intermittently — e.g.
only under parallel test runs.

### The fix (current fork code, `src/open_archive.rs`, ~514-556)

A null/alignment check by itself doesn't close the second hole —
`from_ptr_truncate` still copies a fixed 2048 elements before truncating at
the nul. So the fork doesn't use `from_ptr_truncate` at all for
`UCM_CHANGEVOLUMEW`: it reads the string one character at a time in a loop
that stops at the nul terminator, guaranteeing it never reads past the real
buffer.

```rust
native::UCM_CHANGEVOLUMEW => {
    // libunrar hands us a nul-terminated wide string naming the next
    // volume, but at volume-boundary crossings it can pass null or a
    // non-null pointer not aligned to `WideChar` (u32 on Unix, u16 on
    // Windows). Two hazards must be guarded:
    //   1. Null / misaligned `p` — reading through it is UB.
    //   2. Over-reading. `from_ptr_truncate(p, 2048)` eagerly builds a
    //      2048-element slice and bulk-copies it via
    //      `ptr::copy_nonoverlapping` BEFORE truncating at the nul, so it
    //      reads up to 8 KiB past the (often much shorter) volume-name
    //      buffer. Under the debug pointer checks in current toolchains
    //      that copy aborts the process (SIGABRT), even when `p` itself
    //      is non-null and aligned.
    // Instead, scan element-by-element up to the 2048 bound (unrar's
    // buffer size / max path length since RAR 5.00), stopping at the nul
    // terminator, so we never read past it. If `p` is unusable we skip;
    // libunrar still locates the next volume by path and the
    // `RAR_VOL_ASK => -1` stop path below is preserved.
    let p = p1 as *const widestring::WideChar;
    if !p.is_null() && (p as usize) % std::mem::align_of::<widestring::WideChar>() == 0 {
        let mut chars: Vec<widestring::WideChar> = Vec::new();
        for i in 0..2048usize {
            // SAFETY: `p` is non-null and aligned; each element up to and
            // including the nul terminator lies within unrar's volume-name
            // buffer, and we stop at the terminator without over-reading.
            let ch = unsafe { p.add(i).read() };
            if ch == 0 {
                break;
            }
            chars.push(ch);
        }
        user_data.1 = Some(widestring::WideCString::from_vec_truncate(chars));
    }
    match p2 {
        // Next volume not found. -1 means stop
        native::RAR_VOL_ASK => -1,
        // Next volume found, 0 means continue
        _ => 0,
    }
}
native::UCM_PROCESSDATA => {
    // Guard against null pointer or zero-count callbacks that
    // libunrar may send at volume-boundary crossings.  Creating a
    // Rust reference from a null pointer is always UB (even for
    // zero-length slices), so we skip the call in that case.
    if p1 != 0 && p2 > 0 {
        let raw_slice =
            std::ptr::slice_from_raw_parts(p1 as *const u8, p2 as usize);
        M::process_data(&mut user_data.0, unsafe { &*raw_slice as &_ });
    }
    0
}
```

Both guards are conservative:

- Skipping a null/misaligned `UCM_CHANGEVOLUMEW` only means we don't record
  the next volume's name; libunrar still finds volumes by path, and the
  `RAR_VOL_ASK → -1` stop path is preserved.
- Skipping a null/empty `UCM_PROCESSDATA` chunk never drops real data (a
  genuine chunk always has `p1 != null` and `p2 > 0`).

### Reproducing it

Create a multi-volume RAR (`rar a -ma4 -v2k arc.rar bigfile`) and process the
first volume through the crate's API on a build with debug assertions
(pointer-precondition checks on Rust nightly):

- Without the `UCM_PROCESSDATA` guard — `SIGABRT` when file payload crosses a
  volume boundary (moving into `arc.part2.rar`).
- Without the null/alignment check in `UCM_CHANGEVOLUMEW` — intermittent
  `SIGABRT` on volume change (easier to reproduce with parallel test runs).
- Keeping the null/alignment check but still calling
  `from_ptr_truncate(p, 2048)` instead of the character-by-character scan —
  `SIGABRT` from the over-read is still possible: the pointer is valid, but
  the real volume-name buffer is shorter than 2048 characters, and the
  fixed-length copy runs past its end.

Only with both guards in place — and the character-by-character scan instead
of `from_ptr_truncate` in `UCM_CHANGEVOLUMEW` — do listing and extraction
work correctly.

`unrar_sys` stays on 0.5.8, unchanged — only the safe Rust wrapper is
different.
