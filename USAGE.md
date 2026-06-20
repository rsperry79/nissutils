# nissutils — USAGE

## Purpose

nissutils is a collection of command-line tools for working with Nissan ECU
ROMs (roughly 2000–2017). They handle checksum correction, encryption,
key derivation, binary analysis, and ROM identification.

All tools are standalone executables — no install required.

---

## Tools

### nisrom — ROM analysis

Identify and validate a Nissan ROM dump. Reports ECU ID, Cal ID, checksum
locations, and ROM structure.

```
nisrom <ROMFILE> [OPTIONS]

OPTIONS:
  -v    human-readable output (default)
  -c    CSV output
  -l    CSV headers (can be combined with -c)
  -h    show help
```

**Examples:**

```
nisrom rom.bin
nisrom rom.bin -c -l    # CSV with headers — pipe to a spreadsheet
```

---

### nisckfix1 — Fix checksum (correction-value strategy)

Rewrites the three correction u32 values that bring the ROM checksum into
agreement. Use when the checksum block address is known.

```
nisckfix1 <&cks_s> <&cks_x> <&corrections> <in_file> [<out_file>]
```

If `out_file` is omitted, output goes to `temp.bin`.

**Example:**

```
nisckfix1 757c 7574 0xbfff4 patched.bin fixed.bin
```

---

### nisckfix2 — Fix checksum (rewrite-values strategy)

Alternative checksum fix that directly overwrites the checksum storage
locations rather than adjusting correction values.

```
nisckfix2 <&cks_s> <&cks_x> <in_file> [<out_file>]
```

**Example:**

```
nisckfix2 757c 7574 patched.bin fixed.bin
```

---

### nisdec1 — Decrypt ROM / data file

Decrypt a file using the Nissan NPT_DDL algorithm.

```
nisdec1 <scode> <in_file> [<out_file>]
```

`scode` is a uint32 hex value (the seed/key).

**Example:**

```
nisdec1 55AA00FF encrypted.bin decrypted.bin
```

---

### nisenc1 — Encrypt ROM / data file

Encrypt a file using Nissan Algo 01.

```
nisenc1 <scode> <in_file> [<out_file>]
```

**Example:**

```
nisenc1 55AA00FF plain.bin encrypted.bin
```

---

### nisguess — Derive key from known enc/dec pair

List possible keys that encode a given pair of uint32 values. Useful when
you have one known plaintext/ciphertext pair.

```
nisguess <enc> <dec>
```

Both arguments are uint32 hex values.

---

### nisguess2 — Guess key from enc/dec file pair

Attempt to derive the Nissan encryption key from a pair of encrypted and
decrypted files.

```
nisguess2 <enc_file> <dec_file>
```

---

### findcallargs — Find function calls by r4 argument

Scan a SH2 ROM binary for calls to a target function where register r4
holds a specific value. Useful for tracing how a function is invoked.

```
findcallargs <tgt> <r4val> <in_file>
```

**Example:**

```
findcallargs 0x56738 0x66 rom.bin
```

---

### findrefs — Find memory accesses to an address

Scan a SH2 ROM binary for instructions that read or write a specific
address. Useful for tracking down what code touches a known register or
memory location.

```
findrefs <in_file> <tgt> [<minbase>]
```

`minbase` filters out accesses below a base address (e.g. to skip ROM
self-references and focus on peripheral I/O).

**Example:**

```
findrefs rom.bin 0xffff40ff 0xffff4000
```

---

### unpackdat — Unpack Nissan .dat payload

Unpack a Nissan `.dat` firmware file, extracting the embedded payload.

```
unpackdat <file.dat> <out.dat>
```

---

## AI Summary & Gotchas

**For AI assistants using nissutils:**

- `nisrom` is the right first step when analysing an unknown ROM — it
  identifies ECU family, Cal ID, and checksum block locations without
  modifying the file.
- Use `nisckfix2` after any ROM edit to update checksums before flashing.
  `nisckfix1` is an alternative for ROMs where the correction block address
  is known and you want to preserve the correction-value approach.
- `findcallargs` and `findrefs` operate on raw SH2 binary at file offset
  (Ghidra loads at base 0x0, hardware ROM base is 0xFFFC0000). Always
  confirm whether an address is a file offset or VA before passing it.
- `nisguess` / `nisguess2` require at least one known plaintext/ciphertext
  pair. They are brute-force tools — runtime scales with key space.
- All tools output to `temp.bin` when no `out_file` is given. Never let
  `temp.bin` accumulate in a directory with original ROM files.
