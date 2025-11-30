## SPACE PIRATES 2 - PatriotCTF

### Advanced Pirate Cipher — Treasure Map Decryption

After completing Level 1, the pirates upgraded their encryption system. The Level 2 challenge requires us to find a 32‑byte input which, after being processed by the program, must match the provided `TARGET` array. All transformations in the binary are deterministic and reversible, which means the correct strategy is to invert the cipher step by step.

---

### Program Analysis

The program applies six transformations in order:

1. XOR with a rotating 5‑byte key
2. Rotate-left using a repeating 7-value rotation pattern
3. Swap each adjacent byte pair
4. Subtract the constant `0x5D`
5. Reverse bytes in chunks of 5
6. XOR each byte with `i² mod 256` based on its index

Each transformation is invertible. There is no hashing, randomness, or irreversible component. The binary simply checks whether the final buffer equals `TARGET`. Therefore the optimal path is:

**Apply all inverse operations to `TARGET` in reverse order.**

---

### Inverse Strategy

Reverse order and inverse of each step:

6. XOR again with `i² mod 256`
7. Reverse each 5‑byte chunk again
8. Add `0x5D` instead of subtracting
9. Swap adjacent bytes again
10. Rotate-right by the same rotation pattern
11. XOR with the rotating key again

Once all inverse steps have been applied, the result is the exact 32‑byte plaintext the binary expects.

---

### Solution Script

The following Python script performs all inverse operations and reconstructs the valid input:

```python
# Reverse-engineering Space Pirates Treasure Map (Level 2)

TARGET = [
    0x15, 0x5A, 0xAC, 0xF6, 0x36, 0x22, 0x3B, 0x52,
    0x6C, 0x4F, 0x90, 0xD9, 0x35, 0x63, 0xF8, 0x0E,
    0x02, 0x33, 0xB0, 0xF1, 0xB7, 0x69, 0x42, 0x67,
    0x25, 0xEA, 0x96, 0x63, 0x1B, 0xA7, 0x03, 0x0B
]

XOR_KEY = [0x7E, 0x33, 0x91, 0x4C, 0xA5]
ROTATION_PATTERN = [1, 3, 5, 7, 2, 4, 6]
MAGIC_SUB = 0x5D

buffer = TARGET.copy()

# Step 6 inverse: XOR with i^2
for i in range(len(buffer)):
    buffer[i] ^= (i * i) % 256

# Step 5 inverse: reverse each 5-byte chunk
for chunk_start in range(0, len(buffer), 5):
    buffer[chunk_start:chunk_start+5] = reversed(buffer[chunk_start:chunk_start+5])

# Step 4 inverse: add 0x5D
for i in range(len(buffer)):
    buffer[i] = (buffer[i] + MAGIC_SUB) % 256

# Step 3 inverse: swap adjacent bytes
for i in range(0, len(buffer), 2):
    buffer[i], buffer[i+1] = buffer[i+1], buffer[i]

# Step 2 inverse: rotate right
for i in range(len(buffer)):
    r = ROTATION_PATTERN[i % 7]
    buffer[i] = ((buffer[i] >> (r % 8)) | (buffer[i] << (8 - (r % 8)))) & 0xFF

# Step 1 inverse: XOR again with rotating key
for i in range(len(buffer)):
    buffer[i] ^= XOR_KEY[i % 5]

result = bytes(buffer).decode()
print("Correct input:", result)
```

Executing this script produces the exact plaintext required by the program. Supplying it as the argument causes the binary to accept it and reveal the flag.

Final flag:
```PCTF{Y0U_F0UND_TH3_P1R4T3_B00TY}```
