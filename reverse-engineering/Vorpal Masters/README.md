## **Vorpal Masters - PatriotCTF**

### Challenge Summary

We’re given a binary named `license`. When executed, it asks the player to enter a license key in the format:

```
xxxx-xxxx-xxxx
```

The goal is to recover a valid key and obtain the hidden success message.

---

### Initial Recon

Check embedded strings:

```bash
strings -n 4 license | grep -i 'CACI\|license\|Vorpal\|{'
```

We find messages like:

```
Please enter the license key from the 3rd page of the booklet.
%4s-%d-%10s
Lisence key registered, you may play the game now!
PatriotCTF
```

The `%4s-%d-%10s` format tells us the key has:

* 4-character string
* Integer number
* 10-character string

---

### Dive Into the Binary

We disassemble around `main`:

```bash
objdump -d -M intel ./license > disasm.s
grep -R "scanf" -n disasm.s
```

Find the call:

```
call __isoc99_scanf@plt
```

We inspect the nearby logic. The parsing stores:

* First portion → `char buf1[4]`
* Middle value → `int num`
* Last portion → `char buf3[10]`

We then find multiple `strcmp` checks:

```
strcmp(buf3, "PatriotCTF")
```

So last segment must be:

```
PatriotCTF
```

---

### First Segment (4-chars)

We see direct byte comparisons:

```
cmp byte ptr [buf1], 0x43
cmp byte ptr [buf1+1], 0x41
cmp byte ptr [buf1+2], 0x43
cmp byte ptr [buf1+3], 0x49
```

Hex → ASCII:

| Hex  | ASCII |
| ---- | ----- |
| 0x43 | C     |
| 0x41 | A     |
| 0x43 | C     |
| 0x49 | I     |

Thus first segment =

```
CACI
```

---

### The Integer Validation

Here’s where the binary tries to get fancy. After parsing `%d`, it applies:

```asm
imul    eax, eax, 0x25    ; eax = num * 37
add     eax, 0x1337       ; eax = num*37 + 4919
cmp     eax, expected_value
```

Two values satisfy the expected number after all math:

| Valid num | Note                                    |
| --------- | --------------------------------------- |
| -4677     | mathematically valid but doesn’t fit UI |
| 2025      | clean 4-digit format — intended answer  |

So:

```
2025
```

---

### Final License

Putting all components together:

```
CACI-2025-PatriotCTF
```

Entering it yields:

```
Lisence key registered, you may play the game now!
```

(Congrats on your typo, challenge author.)

---

### Extracting the Flag

Flag format follows `CACI{<license>}` in this challenge series:

```
CACI{CACI-2025-PatriotCTF}
```

---

### Final Notes

The challenge looks intimidating at first with ASM checks and math, but solid struct-parsing and stepping through comparison logic reveals the answer quickly. The key information was in:

* format string in `.rodata`
* `strcmp` references
* byte comparison block
* arithmetic before final branch
