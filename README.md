# CVE-2026-112706: CoreMedia bytestream XPC leaks live file descriptors to any sandboxed app

> **Status: FIXED in iOS 27.2 / iPadOS 27.2 (and tvOS/visionOS/watchOS 27.2), released 2026-10-27.**
> **Affected (verified): iOS 26.5 (23F77), iOS 27.0 (24A437), iOS 27.0.1 (24A446).**

This repository documents an **independent discovery and coordinated disclosure** of this issue.

## Timeline

| Date | Event |
|---|---|
| 2026-10-15 | Issue independently discovered and reported to Apple Product Security (report OE11044060567043). |
| 2026-10-27 | Apple ships the fix in iOS/iPadOS/tvOS/visionOS/watchOS 27.2 as **CVE-2026-112706** ("An app may be able to access sensitive user data" — CoreMedia, "a permissions issue was addressed with additional restrictions"). |
| 2026-10-27 | Advisory credits the earlier report: **Adriatik Raci, Rodolphe BRUNETTI (@eisw0lf) of Lupus Nova, Chris Bailey – Short Circuit**. My report was confirmed as a duplicate of a previously known issue; per Apple policy the duplicate received no bounty and no advisory credit. |
| 2026-10-28 | Post-fix publication of this writeup and PoC for the record. |

I am publishing this because the advisory does not reflect my independent report, and the material here may be useful to researchers studying XPC/sandbox design flaws on Apple platforms. Nothing in this repository is zero-day: **the issue is patched on all current releases. Do not test the PoC against devices you do not own.**

## Summary

Every iOS device exposes the launchd Mach service
`com.apple.coremedia.mediaplaybackd.bytestream.xpc`. A sandboxed third-party app —
no entitlements, no permission prompts, no usage-description keys — can connect to it
directly and obtain **live file descriptors for files outside its sandbox**, opened by
the `mediaplaybackd` daemon with its own credentials.

The chain:

1. **Unauthenticated origin handshake.** The FigXPC memory-origin handshake
   (`.rp1` → `.rp2` → `.org`) completes with status 0 for an unprivileged client.
2. **`Curl` create with a client-controlled path.** The create message carries a
   "CreationURL" dictionary with an absolute `file://` path chosen by the client.
   The daemon opens that path itself, never validating it against the requesting
   client's sandbox container.
3. **Live fd in the reply.** The reply's `ByteStreamDetails` dictionary contains a
   live `FileDescriptor` for the opened file. The app dups it and `pread()`s it —
   descriptor-based I/O bypasses the client sandbox's path checks, so the full
   content of the protected file lands in the app container.
4. **Bonus primitives.** The same request shape yields directory fds
   (`fdopendir`/`readdir` enumeration of protected directories) and a file-existence
   oracle (a real-but-forbidden path returns `-12203`, a nonexistent path `-17913`).

**Control evidence:** in the same run, the app's own `open()` of the identical path
fails with `errno 1 (EPERM)`. The daemon-vended fd returns every byte.

**Incomplete mitigation in iOS 27.0:** on iOS 26.5 the auxiliary opcodes `read` / `fdsc`
/ `getL` were also open; iOS 27.0 sealed those (`-16158`) but left the primary
fd-vending path inside the `Curl` create reply fully functional — verified end-to-end
on 27.0 and 27.0.1.

**Readable domain (mapped on both versions, 27 targets probed):** exactly the
media-library directory `/private/var/mobile/Media/iTunes_Control/iTunes/`
(database trio, `PlayCounts.plist`, `VoiceMemos.plist`, per-user ContentTaste plists,
IC-Info device files). User photos, messages, contacts, the TCC permission database,
keychain, Safari data, Health, Wallet passes and other apps' containers were all
refused with `-12203`. The write path was denied on both versions (`-12878`), so the
primitive is read/enumerate/oracle, not write.

## Evidence

All logs are from clean, stock (non-jailbroken) test devices. Unrelated third-party
tool residue in directory listings is replaced with `[redacted]`.

| File | Device / OS | What it shows |
|---|---|---|
| `evidence/wave4_clean_16pro_result.txt` | iOS 26.5 (23F77) | EPERM control, fd steal 749,568 + 2,175,392 B, `read` data channel, dir list (18), oracle |
| `evidence/wave4_ios270_result.txt` | iOS 27.0 (24A437) | `Curl` OK, fd in ByteStreamDetails, steals 835,584 + 1,730,432 + 32,768 B, aux opcodes sealed `-16158`, dir list (13), oracle |
| `evidence/wave4_ios2701_result.txt` | iOS 27.0.1 (24A446) | Re-run after update: `Curl`/fd path still live (835,584 + 1,862,272 B) — **not fixed in 27.0.1** |
| `evidence/wave5_domain_map.txt` | both | 27-target readable-domain map; only `iTunes_Control/iTunes/` opens |
| `evidence/wave4_ab_matrix.md` | both | One-page 26.5 vs 27.0 channel matrix |

The stolen database copies themselves are not published (test-device data); the PoC
regenerates them on any affected device.

## PoC

`poc/` contains a minimal Xcode project (`MConfPoC`). To run:

1. Open in Xcode, set your development team (the project intentionally has
   `YOUR_TEAM_ID` placeholders).
2. Build & run on a device running an affected iOS version (≤ 27.0.1).
3. Launch the app and enter an absolute target path. The target may be a
   regular file or a directory; whether it can be opened is determined by the
   daemon.
4. Press **Run**. A regular file is saved as
   `Documents/Recovered-<filename>` (or `GlobalPreferences.plist` for the
   legacy test path). A directory produces a one-level manifest at
   `Documents/Recovered-<directory>.directory.txt`, containing each immediate
   entry's name, type and reported size (up to 10,000 entries).
5. When a directory is opened, the app also shows an interactive browser:
   **Open** requests the selected child directory and lists its immediate
   entries; **Parent directory** moves one level up; **Select** copies a regular
   file's full path into the target field so it can be read with **Run**. The
   browser does not follow symbolic links and does not recursively crawl a
   directory tree. **Browse last directory** reopens the most recent listing.
   Directory manifests and regular-file results can be shared with
   **Export result**. Short execution logs are shown in the app and saved as
   `Documents/run.log`. The app display name is `CVE-2026-112706`.

Core chain: `dlopen` CoreMedia + `dlsym` the FigXPC helpers →
`xpc_connection_create_mach_service` → `.rp1`/`.rp2`/`.org` handshake →
`FigXPCCreateBasicMessage('Curl')` + `FigXPCMessageSetCFURL("CreationURL")` → reply
`ByteStreamDetails.FileDescriptor` → `fstat` followed by either `pread` for a
regular file or `fdopendir`/`readdir` for a directory.

## Suggested fix (as reported to Apple)

1. On every `Curl` create, verify the requesting process (audit token / sandbox
   ownership) and reject paths the client itself cannot open; never open
   client-supplied absolute paths with daemon credentials.
2. Apply the same gating to the `FileDescriptor` embedded in the create reply as was
   applied to the `read`/`fdsc` opcodes in iOS 27.0.
3. Extend the check to directory opens (enumeration primitive).
4. Return a uniform error for "denied because exists" vs "does not exist" to remove
   the existence oracle.

## References

- Apple — About the security content of iOS 27.2 and iPadOS 27.2 (CVE-2026-112706)
- Apple — About the security content of tvOS 27.2 / visionOS 27.2 / watchOS 27.2 (same CVE)

## Disclaimer

Provided for educational and defensive research purposes only. The vulnerability is
patched; running the PoC on current iOS will simply fail. The author is not
responsible for misuse.
