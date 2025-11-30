## **SPACE PIRATES - PatriotCTF**

### Challenge Summary

We recovered a C program used by space pirates to validate encrypted coordinates. The goal: reverse the transformation logic and recover the original flag that successfully validates against `TARGET`.

The binary performs **four reversible encryption steps** on the input before comparing with a static encrypted byte array.

---

### 🔍 Encryption Stages (forward)

Given a 30-byte input, the program performs:

1. **XOR with rotating 5-byte key**

   ```
   XOR_KEY = {0x42, 0x73, 0x21, 0x69, 0x37}
   ```
2. **Swap every adjacent byte pair**

   ```
   (0,1), (2,3), (4,5), ...
   ```
3. **Add constant (mod 256)**

   ```
   MAGIC_ADD = 0x2A
   ```
4. **XOR with index**

   ```
   buffer[i] ^= i
   ```

The result is compared with a hard-coded encrypted array `TARGET`.

---

### Reversing the Encryption

To retrieve the correct plaintext, we apply inverse steps in reverse order:

| Original Step  | Reverse Action                   |
| -------------- | -------------------------------- |
| XOR with index | XOR with index again             |
| Add constant   | Subtract `0x2A` (mod 256)        |
| Swap pairs     | Swap pairs again                 |
| XOR with key   | XOR again with same rotating key |

The entire reverse process can be automated. For example:

```python
TARGET = [
    0x5A,0x3D,0x5B,0x9C,0x98,0x73,0xAE,0x32,0x25,0x47,
    0x48,0x51,0x6C,0x71,0x3A,0x62,0xB8,0x7B,0x63,0x57,
    0x25,0x89,0x58,0xBF,0x78,0x34,0x98,0x71,0x68,0x59
]

KEY = [0x42, 0x73, 0x21, 0x69, 0x37]
MAGIC_ADD = 0x2A
FLAG_LEN = 30

buf = TARGET.copy()

# Reverse step 4: XOR with index
for i in range(FLAG_LEN):
    buf[i] ^= i

# Reverse step 3: Subtract constant
for i in range(FLAG_LEN):
    buf[i] = (buf[i] - MAGIC_ADD) % 256

# Reverse step 2: Swap back
for i in range(0, FLAG_LEN, 2):
    buf[i], buf[i+1] = buf[i+1], buf[i]

# Reverse step 1: XOR with key
for i in range(FLAG_LEN):
    buf[i] ^= KEY[i % 5]

print("Recovered Flag:", bytes(buf).decode())
```

---

### Final Flag

```
PCTF{0x_M4rks_tH3_sp0t_M4t3ys}
```
