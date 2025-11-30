## SPACE PIRATES 3 - PatriotCTF

The third stage of the Space Pirates series raises the stakes: Captain Blackbyte claims this vault is protected by “the most devious cipher ever created by pirate‑kind.” Underneath the theatrics, though, the program still performs a fully reversible sequence of byte‑level transformations. That means the path to the treasure is the same as before: invert every step the binary performs, in reverse order, until the original 30‑byte plaintext emerges.

The binary reads a 30‑byte input, applies six deterministic operations, and compares the result to a 30‑byte `target` array. No randomness, no hashing, no one‑way functions. Everything is mathematically invertible. So the correct strategy is simply to unwind the pipeline.

---

### Program Analysis

The encryption chain consists of six transformations:

1. XOR with a rotating 7‑byte key
2. Bit rotate‑left using an 8‑value rotation pattern (including rotation by 0)
3. Swap each adjacent byte pair
4. Subtract the constant `0x93`
5. Reverse bytes in chunks of 6
6. XOR each byte with `(i² + i) mod 256` based on its index

Every operation is bijective. XOR is self‑inverse, swapping pairs is self‑inverse, reversing chunks is self‑inverse, rotating left can be undone by rotating right, and subtracting a constant can be undone by adding it. Because of this, the full cipher is also bijective.

The binary’s job is simple: after these six steps, the transformed input must equal `target`.

Thus the solution method is:

**Apply the inverse of each transformation to `target`, in the exact reverse order.**

---

### Inverse Strategy

Reversing the order and applying the correct inverse gives us:

6. XOR again with `(i² + i) mod 256`
7. Reverse the 6‑byte chunks again
8. Add `0x93` instead of subtracting
9. Swap adjacent bytes again
10. Rotate‑right using the same rotation pattern
11. XOR again with the same 7‑byte key

Running these six inverse operations yields the original 30‑byte vault combination.

---

### Solution Script

The Python script below performs all inverse steps and reconstructs the exact plaintext expected by the program:

```python
# Reverse-engineering Space Pirates Vault (Level 3)

from typing import List

target = [
    0x60, 0x6D, 0x5D, 0x97, 0x2C, 0x04, 0xAF, 0x7C, 0xE2, 0x9E,
    0x77, 0x85, 0xD1, 0x0F, 0x1D, 0x17, 0xD4, 0x30, 0xB7, 0x48,
    0xDC, 0x48, 0x36, 0xC1, 0xCA, 0x28, 0xE1, 0x37, 0x58, 0x0F
]

xor_key = [0xC7, 0x2E, 0x89, 0x51, 0xB4, 0x6D, 0x1F]
rotation_pattern = [7, 5, 3, 1, 6, 4, 2, 0]
magic_sub = 0x93
chunk_size = 6

def ror(value, n):
    n = n % 8
    return ((value >> n) | ((value << (8 - n)) & 0xFF)) & 0xFF

buf = target.copy()

# Step 6 inverse: XOR with (i^2 + i)
for i in range(len(buf)):
    pos_val = ((i * i) + i) % 256
    buf[i] ^= pos_val

# Step 5 inverse: reverse 6-byte chunks
for start in range(0, len(buf), chunk_size):
    buf[start:start+chunk_size] = reversed(buf[start:start+chunk_size])

# Step 4 inverse: add 0x93
for i in range(len(buf)):
    buf[i] = (buf[i] + magic_sub) & 0xFF

# Step 3 inverse: swap adjacent bytes
for i in range(0, len(buf), 2):
    buf[i], buf[i+1] = buf[i+1], buf[i]

# Step 2 inverse: rotate-right
for i in range(len(buf)):
    r = rotation_pattern[i % len(rotation_pattern)]
    buf[i] = ror(buf[i], r)

# Step 1 inverse: XOR again with 7-byte key
for i in range(len(buf)):
    buf[i] ^= xor_key[i % len(xor_key)]

result = bytes(buf).decode()
print("Correct vault combination:", result)
```

Executing this produces the exact plaintext accepted by the Level 3 binary:

```
PCTF{M4ST3R_0F_TH3_S3V3N_S34S}
```

This input unlocks the vault and prints the final flag.

Final flag:

```
PCTF{M4ST3R_0F_TH3_S3V3N_S34S}
```
