---
title: Tools
icon: fas fa-wrench
order: 5
---

## Tools

Small projects I built while learning. These are practice tools, not production software.

### [zip-parser](https://github.com/JayasuryaNair01/zip-parser) (in progress)

A ZIP parser in Rust, written from scratch with no ZIP libraries, to understand the format at the byte level. It finds the End of Central Directory record, reads the central directory and local headers, and extracts stored (uncompressed) files. Next: all entries, Deflate, bounds-checked reads and zip slip protection.

### [PassGuardian](https://github.com/JayasuryaNair01/PassGuardian)

A Python CLI that scores a password with basic local rules, optionally checks a wordlist, and looks it up in HaveIBeenPwned using the k-anonymity range API (only the first 5 characters of the SHA-1 hash leave the machine).

### [ReconLite](https://github.com/JayasuryaNair01/ReconLite)

A small multi-threaded recon CLI in Python for practice: subdomain brute-force, directory discovery, and a basic TCP port scan. Use only on targets you're allowed to test.

### [ytm-play](https://github.com/JayasuryaNair01/ytm-play)

A terminal YouTube Music search-and-play client in Rust. I wrote it to learn async networking (`tokio` / `reqwest`) and nested JSON parsing. Playback goes through `mpv`.

