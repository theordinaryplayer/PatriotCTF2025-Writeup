
# Matrix Reconstruction — PatriotCTF

---

## Challenge Overview

We are given:

- **cipher.txt** — the encrypted flag.
- **keystream_leak.txt** — a list of 32-bit integers representing internal states of a pseudorandom generator.

The PRG is based on a linear recurrence in **GF(2)**:

\[
S_{n+1} = A \cdot S_n \oplus B
\]

Where:

- \( S_n \) = 32-bit state vector
- \( A \) = unknown 32×32 binary matrix
- \( B \) = unknown 32-bit constant vector
- \( \cdot \) = matrix multiplication over GF(2)
- \( \oplus \) = bitwise XOR

The **keystream byte** is the **lowest byte** of \( S_n \), i.e., \( S_n \ \& \ 0xFF \).

Our task:  
Recover \( A \) and \( B \) from the leaked states, then decrypt `cipher.txt` to get the flag.

---

##   Step 1 — Understanding the math

We have:

\[
S_{n+1} = A S_n \oplus B
\]
\[
S_{n+2} = A S_{n+1} \oplus B
\]

Subtracting (XOR) the two equations:

\[
S_{n+2} \oplus S_{n+1} = A (S_{n+1} \oplus S_n)
\]

Let:

\[
\Delta_n = S_{n+1} \oplus S_n
\]

Then:

\[
\Delta_{n+1} = A \cdot \Delta_n
\]

This **eliminates \( B \)** completely.

---

## Step 2 — Solving for \( A \)

We can collect:

\[
X = [ \Delta_0 \ \Delta_1 \ \dots \ \Delta_{31} ]
\]
\[
Y = [ \Delta_1 \ \Delta_2 \ \dots \ \Delta_{32} ]
\]

Then:

\[
Y = A X \quad \Rightarrow \quad A = Y X^{-1}
\]

All operations are in **GF(2)**.

We just need 32 linearly independent \( \Delta_n \) vectors to ensure \( X \) is invertible.

---

## Step 3 — Recovering \( B \)

Once \( A \) is known:

\[
B = S_{n+1} \oplus A S_n
\]

We can use any \( n \) to compute \( B \).

---

## Step 4 — Implementation details

1. **Parse** leaked states as integers.
2. **Compute** \( \Delta_n = S_{n+1} \oplus S_n \).
3. **Build** \( X \) and \( Y \) as 32×32 binary matrices (each column = bits of \( \Delta_n \)).
4. **Invert** \( X \) over GF(2) using Gaussian elimination.
5. **Compute** \( A = Y X^{-1} \).
6. **Compute** \( B = S_1 \oplus A S_0 \).
7. **Verify** with another state to ensure correctness.
8. **Decrypt** ciphertext:
   - Start from \( S_0 \) in leaked states.
   - Generate next state: \( S_{n+1} = A S_n \oplus B \).
   - Keystream byte = low byte of \( S_n \).
   - XOR with ciphertext.

---

## Step 5 — Python code

```python
import numpy as np

# Load leaked states
with open("keystream_leak.txt", "r") as f:
    states = [int(x) for x in f.read().split()]

N = len(states)
print(f"Loaded {N} states")

# Compute D_n = S_{n+1} xor S_n
D = [states[i+1] ^ states[i] for i in range(N-1)]

# Build X and Y matrices over GF(2)
X_bits = np.zeros((32, 32), dtype=np.uint8)
Y_bits = np.zeros((32, 32), dtype=np.uint8)

for i in range(32):
    x_vec = D[i]
    y_vec = D[i+1]
    for b in range(32):
        X_bits[b, i] = (x_vec >> b) & 1
        Y_bits[b, i] = (y_vec >> b) & 1

# Invert X over GF(2)
def gf2_inv(matrix):
    n = matrix.shape[0]
    mat = matrix.astype(int)
    inv = np.eye(n, dtype=int)
    # Forward elimination
    for i in range(n):
        if mat[i, i] == 0:
            for j in range(i+1, n):
                if mat[j, i] == 1:
                    mat[[i, j]] = mat[[j, i]]
                    inv[[i, j]] = inv[[j, i]]
                    break
        for j in range(i+1, n):
            if mat[j, i] == 1:
                mat[j] ^= mat[i]
                inv[j] ^= inv[i]
    # Backward elimination
    for i in range(n-1, -1, -1):
        for j in range(i-1, -1, -1):
            if mat[j, i] == 1:
                mat[j] ^= mat[i]
                inv[j] ^= inv[i]
    return inv

X_inv = gf2_inv(X_bits)
A = (Y_bits @ X_inv) % 2

# Recover B
def bits_to_int(bits):
    return sum((bits[i] << i) for i in range(32))

S0 = states[0]
S1 = states[1]
S0_bits = [(S0 >> i) & 1 for i in range(32)]
AS0_bits = A @ S0_bits
AS0 = bits_to_int(AS0_bits % 2)
B = S1 ^ AS0

print("B =", B)

# Verify
S2 = states[2]
S1_bits = [(S1 >> i) & 1 for i in range(32)]
AS1_bits = A @ S1_bits
AS1 = bits_to_int(AS1_bits % 2)
assert (AS1 ^ B) == S2
print("A and B verified successfully")

# Decrypt
with open("cipher.txt", "rb") as f:
    ciphertext = f.read()

keystream = []
s = states[0]
for _ in range(len(ciphertext)):
    keystream.append(s & 0xFF)
    s_bits = [(s >> i) & 1 for i in range(32)]
    s_next_bits = A @ s_bits
    s_next = bits_to_int(s_next_bits % 2)
    s = s_next ^ B

plaintext = bytes(c ^ k for c, k in zip(ciphertext, keystream))
print(plaintext.decode())
```

---

## Step 6 — Flag

Running the script produces the decrypted message containing the flag:

```
pctf{mAtr1x_r3construct?on_!s_fu4n}
```

---

## Conclusion

The vulnerability here was using a **linear recurrence in GF(2)** without enough security — knowing consecutive states allows full recovery of \( A \) and \( B \).  
This is a classic case where **linearity** in crypto leads to breaks.

**Lesson**: Never use purely linear transformations for stream ciphers without non-linear components.
