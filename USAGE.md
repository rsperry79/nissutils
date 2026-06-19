# nissutils — USAGE

## Purpose

A collection of CLI utilities for working with Nissan ECU ROM files.
Covers ROM analysis, checksum repair, encryption/decryption, key recovery,
and SH2 binary reverse-engineering tasks. All tools operate on raw binary
files and are designed to be scripted together in a workflow.

---

## Tools

### `nisrom` — Analyze a Nissan ROM

Parses a Nissan ECU ROM binary and extracts metadata: ECU ID, FID, loader
version, checksum addresses, RAM function offset, IVT2 location, and
checksum validity. Output can be human-readable or CSV for scripting.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<ROMFILE>` | Yes | Path to the ROM binary |
| `-v` | No | Human-readable output (default if no flags given) |
| `-c` | No | CSV value output |
| `-l` | No | Print CSV header row (combine with `-c` for full CSV) |
| `-f` | No | Force parsing even if errors detected (risk of crash) |
| `-h` | No | Show help |

**Examples:**
```bash
# Human-readable summary
nisrom ROM_MECM07-370c1-512.bin

# Full CSV with header (useful for scripting)
nisrom -l -c ROM_MECM07-370c1-512.bin

# Collect checksum addresses for nisckfix2
nisrom ROM_MECM07-370c1-512.bin | grep -i checksum
```

---

### `nisckfix1` — Fix checksum with correction pointer

Recalculates and writes a ROM checksum using a *corrections* location.
Use when the ROM has a dedicated region for storing fix-up bytes.
Requires three hex addresses: the sum pointer, XOR pointer, and the
corrections block pointer.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<&cks_s>` | Yes | Hex offset of the sum checksum field |
| `<&cks_x>` | Yes | Hex offset of the XOR checksum field |
| `<&corrections>` | Yes | Hex offset of the corrections block |
| `<in_file>` | Yes | Input ROM binary |
| `[out_file]` | No | Output file (default: `temp.bin`) |

**Example:**
```bash
nisckfix1 7578 7574 757c hackrom.bin patched.bin
```

---

### `nisckfix2` — Fix checksum in-place

Recalculates the ROM sum and XOR checksums and writes them directly into
the ROM at the given addresses. Simpler than nisckfix1; no corrections
pointer needed. Use when you know the two checksum field addresses.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<&cks_s>` | Yes | Hex offset of the sum checksum field |
| `<&cks_x>` | Yes | Hex offset of the XOR checksum field |
| `<in_file>` | Yes | Input ROM binary |
| `[out_file]` | No | Output file (default: `temp.bin`) |

**Example:**
```bash
nisckfix2 757c 7574 hackrom.bin fixed.bin
```

---

### `nisdec1` — Decrypt ROM (NPT_DDL algorithm)

Decrypts a Nissan ROM or payload file using the NPT_DDL stream cipher.
The seed code must match what the ECU used to encrypt. Required before
patching certain encrypted ROM images.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<scode>` | Yes | 32-bit hex seed/key (e.g. `55AA00FF`) |
| `<in_file>` | Yes | Encrypted input file |
| `[out_file]` | No | Decrypted output (default: `temp.bin`) |

**Example:**
```bash
nisdec1 55AA00FF encrypted.bin decrypted.bin
```

---

### `nisenc1` — Encrypt file (Algo 01)

Encrypts a file using Nissan's Algorithm 01 stream cipher. Inverse of
`nisdec1`. Also outputs the 16-bit checksum of the encrypted result.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<scode>` | Yes | 32-bit hex seed/key |
| `<in_file>` | Yes | Plaintext input file |
| `[out_file]` | No | Encrypted output (default: `temp.bin`) |

**Example:**
```bash
nisenc1 55AA00FF plaintext.bin encrypted.bin
```

---

### `nisguess` — Brute-force key from one enc/dec pair

Given a single known encrypted/decrypted 32-bit value pair, brute-forces
the Nissan encryption key. Fast when you have a known plaintext word
(e.g. a ROM header constant).

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<enc>` | Yes | Encrypted value as hex uint32 |
| `<dec>` | Yes | Known decrypted value as hex uint32 |

**Example:**
```bash
nisguess AABBCCDD 11223344
```

---

### `nisguess2` — Brute-force key from enc/dec file pair

Given two files (one encrypted, one decrypted), brute-forces the key by
testing all candidates against both files simultaneously. More reliable
than `nisguess` when you have full file pairs.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<enc_file>` | Yes | Encrypted binary file |
| `<dec_file>` | Yes | Known plaintext binary file |

**Example:**
```bash
nisguess2 encrypted.bin known_plain.bin
```

---

### `unpackdat` — Unpack Nissan .dat payload

Extracts the payload from a Nissan ECU `.dat` transfer format file.
The `.dat` format wraps binary data with framing, CRC, and checksum bytes
in a specific packet structure. Writes the raw binary payload to the
output file.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<file.dat>` | Yes | Input `.dat` file |
| `<out.dat>` | Yes | Output unpacked binary |

**Example:**
```bash
unpackdat firmware_update.dat payload.bin
```

---

### `findrefs` — Find SH2 references to an address

Scans a raw SH2 binary for all instructions that reference a given target
address via base+displacement addressing. Reports the offset of each
instruction and whether it is a read or write access. Useful for locating
all code that accesses a known RAM or I/O address.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<in_file>` | Yes | Raw SH2 ROM binary |
| `<tgt>` | Yes | Target address to search for (hex) |
| `[minbase]` | No | Minimum base register value to consider (filters noise) |

**Example:**
```bash
findrefs rom.bin 0xffff40ff 0xffff4000
```

---

### `findcallargs` — Find call sites passing a specific argument

Searches a raw SH2 binary for all call sites of a function (`<tgt>`)
where register R4 is loaded with a specific value (`<r4val>`) before
the call. Used to find all callers that pass a known argument — e.g.
all DTC raise calls with a specific fault code.

**Args:**
| Arg | Required | Description |
|-----|----------|-------------|
| `<tgt>` | Yes | Function address (hex) |
| `<r4val>` | Yes | Expected R4 value at the call site (hex) |
| `<in_file>` | Yes | Raw SH2 ROM binary |

**Example:**
```bash
findcallargs 0x56738 0x66 rom.bin
```

---

## AI Summary & Gotchas

**For AI assistants using these tools:**

- All addresses and values are **hexadecimal without `0x` prefix** unless the
  tool explicitly shows `0x` in its own examples (`findcallargs` / `findrefs`
  accept both).
- `nisrom` is the **starting point** for any ROM workflow — run it first to
  get checksum offsets before calling `nisckfix1`/`nisckfix2`.
- `nisckfix1` vs `nisckfix2`: use `nisckfix1` when the ROM has a dedicated
  corrections block (older MEC07-series); use `nisckfix2` when you can write
  directly to the two checksum fields.
- **Never patch the original ROM.** Work on a copy; these tools overwrite in
  the output file.
- `nisdec1` and `nisenc1` are **not symmetric** in algorithm — `nisdec1` uses
  NPT_DDL, `nisenc1` uses Algo 01. Verify which algorithm applies to your ECU
  before using.
- `nisguess` / `nisguess2` can take significant time depending on key space
  and hardware. `nisguess2` is preferred when you have both enc and dec files
  as it eliminates false positives faster.
- `findrefs` and `findcallargs` operate on **raw binary at offset 0** — all
  reported addresses are file offsets, not virtual addresses. On SH7055 the
  ROM maps to `0xFFFC0000`, so add that base if cross-referencing with a
  Ghidra project loaded at the hardware VA.
- `unpackdat` writes a debug log alongside the output; expect a
  `nisrom_dbg.log` or similar file after each run.
