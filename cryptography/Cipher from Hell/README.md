
# Eighth Circle of Hell - PatriotCTF

**Challenge description:**  
We've recovered an encrypted message from the eighth circle of Hell (Malbolge reference). The encryption uses a custom base-3 transformation with a substitution matrix. Can you recover the hidden flag?

---

## Step 1: Understanding the Encryption

The encryption script takes the input flag as bytes, converts it to a big integer `s`, then processes it in base 3.

Key components:

1. **Base-3 requirement:**  
   The highest power of 3 in `s` (i.e., `c = floor(log3(s))`) must be odd, meaning the number of base-3 digits is even.

2. **Transformation matrix `o`:**
   ```python
   o = (
        (6, 0, 7),
        (8, 2, 1),
        (5, 4, 3)
   )
   ```

3. **Encoding process:**  
   - Process digits from the outside in: take the first and last base-3 digits of current `s` as `(d1, d2)`
   - Use `o[d1][d2]` to get a base-9 digit (0–8)
   - Build `ss` in base 9 from these digits
   - Remove first and last base-3 digits from `s`
   - Repeat until all digits processed

4. **Output:**  
   `ss` is written to `encrypted` as raw bytes.

---

## Step 2: Reversing the Process

To decrypt, we need to:

1. Read `encrypted` → integer `ss`
2. Convert `ss` to base-9 digits
3. Reverse the substitution using the inverse of matrix `o`
4. Reconstruct base-3 digits from the pairs
5. Convert base-3 number back to bytes

---

## Step 3: Building the Decoder

### Inverse Mapping
We create a reverse lookup table from the `o` matrix:

```
0 → (0,1)
1 → (1,2)
2 → (1,1)
3 → (2,2)
4 → (2,1)
5 → (2,0)
6 → (0,0)
7 → (0,2)
8 → (1,0)
```

### Reconstruction
If base-9 digits are `[g0, g1, g2, ...]`, then:
- `g0 = o[a0][a_{n-1}]`
- `g1 = o[a1][a_{n-2}]`
- etc.

So we can fill the base-3 array symmetrically:
- `base3[i] = d1` from reverse lookup of `gi`
- `base3[n-1-i] = d2` from reverse lookup of `gi`

---

## Step 4: Implementation

```python
import math

# Read encrypted file
with open('encrypted', 'rb') as f:
    encrypted_bytes = f.read()
ss = int.from_bytes(encrypted_bytes)

# Reverse mapping from o
o = [
    [6, 0, 7],
    [8, 2, 1],
    [5, 4, 3]
]

rev = {}
for i in range(3):
    for j in range(3):
        rev[o[i][j]] = (i, j)

# Convert ss to base 9 digits
def to_base9(n):
    digits = []
    while n > 0:
        digits.append(n % 9)
        n //= 9
    return digits[::-1]  # most significant first

base9_digits = to_base9(ss)
n = len(base9_digits) * 2

# Reconstruct base3 digits
base3 = [None] * n
for i in range(len(base9_digits)):
    d1, d2 = rev[base9_digits[i]]
    base3[i] = d1
    base3[n - 1 - i] = d2

# Convert base3 digits to integer
s = 0
for d in base3:
    s = s * 3 + d

# Convert to bytes
def to_bytes(n):
    byte_length = (n.bit_length() + 7) // 8
    return n.to_bytes(byte_length, 'big')

flag_bytes = to_bytes(s)
print(flag_bytes.decode(errors='ignore'))
```

---

## Step 5: Getting the Flag

Running this decoder on the provided `encrypted` file gives the flag:

```
pctf{a_l3ss_cr4zy_tr1tw1s3_op3r4ti0n_f37d4b}
```

---

## Conclusion

The challenge used a clever base-3 to base-9 transformation with a fixed substitution matrix. The key insight was recognizing the symmetric digit pairing and building the appropriate reverse mapping. The Malbolge reference was mainly thematic — the actual cryptography was a custom encoding scheme rather than Malbolge's infamous complexity.
