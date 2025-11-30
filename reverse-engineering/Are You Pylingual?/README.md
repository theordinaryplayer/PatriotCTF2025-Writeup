## Are You Pylingual? - PatriotCTF

### Challenge Summary

We’re given a Python script and a giant list of integers printed as output. Digging through the script shows that the flag is **not** directly encoded into that scary number array… instead, it’s subtly *inserted* into an ASCII-art string before encryption.

This is one of those “the real trick is in the middle of the mess” challenges.

---

## **What the Script Does**

Breakdown of the relevant operations:

1. Reads `flag.txt` → stores in `flag`
2. Creates big ASCII-art text using:

   ```
   words = 'MASONCC IS THE BEST CLUB EVER'
   art = list(pyfiglet.figlet_format(words, font='slant'))
   ```
3. It loops through indexes of that ASCII art:

   * Starting at `i = len(art) % 10`
   * Every +28 characters:
     → replaces that ASCII-art character with the *next character of the flag*
4. Joins the art together into `art_str`
5. Splits into two halves: `first_half`, `second_half`
6. Each half gets encoded separately:

   ```
   first  -> [~ord(c) ^ 5]
   second -> [~ord(c) ^ 6]
   ```
7. Output = second + first
   (order intentionally swapped)

So the flag lived **in plaintext** inside the ASCII-art *before* any encoding.

---

## That Giant Output Is a Trap

Reversing the whole bitwise obfuscation would work…
but we noticed something smarter:

Since the flag characters were placed in very predictable positions (every 28th char), we can just **recreate the ASCII-art and pull the characters back out**.

---

## Solution Script

```python
import pyfiglet

words = 'MASONCC IS THE BEST CLUB EVER'
font = 'slant'
art = list(pyfiglet.figlet_format(words, font=font))

flag = []

# Same logic as original
i = len(art) % 10
while i < len(art):
    flag.append(art[i])
    i += 28

print(''.join(flag))
```

### Output

```
pctf{obFusc4ti0n_i5n't_EncRypt1oN}
```

---

## Summary

* Obfuscation ≠ encryption
* Reverse order: always inspect earlier logic first
* Recognize predictable insertion patterns
* ASCII art isn’t always innocent…
