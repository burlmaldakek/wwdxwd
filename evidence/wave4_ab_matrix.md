# CoreMedia bytestream PoC — version A/B channel matrix

| Channel | iOS 26.5 (23F77) — iPhone 16 Pro | iOS 27.0 (24A437) — iPhone 17 Pro |
|---|---|---|
| Connect to `com.apple.coremedia.mediaplaybackd.bytestream.xpc` (no entitlement) | OK | OK |
| Origin handshake `'.rp1'` → `'.rp2'` → `'.org'` (unprivileged client) | all st = 0 | all st = 0 |
| Direct `open()` of the target path by the app itself (control) | errno 1 EPERM (denied) | errno 1 EPERM (denied) |
| `'Curl'` create with client-chosen `CreationURL` | OK | OK |
| **`ByteStreamDetails["FileDescriptor"]` (live fd in create reply)** | **✔ delivered** | **✔ still delivered (main channel open)** |
| fd dup + `fstat`/`pread` → full file copy into app container | 749,568 B + WAL 2,175,392 B | 835,584 B + WAL 1,730,432 B + SHM 32,768 B |
| `'read'` opcode (raw data in XPC reply) | ✔ returns data (n=64) | ✘ sealed, error -16158 |
| `'fdsc'` opcode (second fd channel) | ✔ fd | ✘ sealed, error -16158 |
| `'getL'` opcode (size query) | ✔ | ✘ sealed, error -16158 |
| `'cpy'` opcode | ✘ -12784 | ✘ -16158 |
| `'doom'` (close) | ✔ | ✘ -16158 |
| Directory open → directory fd (mode 40755) → `fdopendir`/`readdir` listing | ✔ 18 entries | ✔ 13 entries |
| Existence oracle: -12203 (exists, refused) vs -17913 (ENOENT) | ✔ (protected dir real vs fake) | ✔ (generalizes across DCIM/probe set) |
| Write path (`Curl` with write open options) | ✘ -12878 (denied by FigServer write gate) | ✘ -12878 (denied) |

## Reading

- The **primary primitive — a live file descriptor embedded in the `'Curl'` create reply — is open on both versions.** The iOS 27.0 mitigation sealed the auxiliary opcodes (`'read'`, `'fdsc'`, `'getL'`) with error -16158 but did **not** seal the create-reply fd channel: the same chain that works on 26.5 still exfiltrates the full database trio on 27.0.
- Same-app control on both: a direct `open()` of the same paths returns EPERM — the fd vended by the daemon is the actual access vector.
- Write remains denied on both versions (-12878); this is a read/enumeration/oracle primitive.
- Bounded by the daemon's own privilege domain: paths the daemon itself cannot open (e.g. daemon-inaccessible protected paths) return -12203 to the client — but the -12203/-17913 distinction itself constitutes a filesystem existence oracle for arbitrary absolute paths.

Logs: `wave4_clean_16pro_result.txt` (26.5), `wave4_ios270_result.txt` (27.0).
