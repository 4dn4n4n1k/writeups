# PicoCTF — Text Transformations

**Category:** General Skills / Scripting
**Connection:** `nc foggy-cliff.picoctf.net 58409`
**Flag:** `picoCTF{Revers1ng_t3xt_Tr4nsf0rm@t10ns_3a939318}`

## Challenge

The server applies a chain of Linux text transformations to the flag and asks you, step by step, to enter the *single Linux command* that reverses each transformation. Get one wrong and the connection closes.

```
===Welcome to the Text Transformations Challenge!===
--- Step 1 ---
Current flag: KTgxMzkzOW4zLWZhMDFnQHplMHNmYTRlRy1nazNnLXRhMWZlcmlyRShTR1BicHZj
Hint: Base64 encoded the string.
Enter the Linux command to reverse it:
```

## Approach

Rather than typing each command manually under a possible time limit, I scripted the interaction with `pwntools` and dispatched commands based on the hint string. The chain happened to be only 5 steps, but the script generalises to any tr-style swap.

## Step-by-step

| # | Input                                                              | Hint                                  | Reverse command              |
|---|--------------------------------------------------------------------|---------------------------------------|------------------------------|
| 1 | `KTgxMzkzOW4zLWZhMDFnQHplMHNmYTRlRy1nazNnLXRhMWZlcmlyRShTR1BicHZj` | Base64 encoded                        | `base64 -d`                  |
| 2 | `)813939n3-fa01g@ze0sfa4eG-gk3g-ta1ferirE(SGPbpvc`                 | Reversed the text                     | `rev`                        |
| 3 | `cvpbPGS(Eriref1at-g3kg-Ge4afs0ez@g10af-3n939318)`                 | Replaced underscores with dashes      | `tr - _`                     |
| 4 | `cvpbPGS(Eriref1at_g3kg_Ge4afs0ez@g10af_3n939318)`                 | Replaced curly braces with parens     | `tr '()' '{}'`               |
| 5 | `cvpbPGS{Eriref1at_g3kg_Ge4afs0ez@g10af_3n939318}`                 | Applied ROT13 to letters              | `tr A-Za-z N-ZA-Mn-za-m`     |

After step 5 the server prints:

```
Congratulations! You've recovered the original flag:
picoCTF{Revers1ng_t3xt_Tr4nsf0rm@t10ns_3a939318}
```

## Solver

```python
#!/usr/bin/env python3
from pwn import *

io = remote('foggy-cliff.picoctf.net', 58409)

def step():
    data = io.recvuntil(b'Enter the Linux command to reverse it:', timeout=10).decode()
    print(data)
    hint = [l for l in data.splitlines() if l.startswith('Hint:')][-1].lower()
    return hint

while True:
    try:
        h = step()
    except EOFError:
        break
    if 'base64' in h:                              cmd = 'base64 -d'
    elif 'rot13' in h or 'caesar' in h:            cmd = 'tr A-Za-z N-ZA-Mn-za-m'
    elif 'reverse' in h:                           cmd = 'rev'
    elif 'underscore' in h and 'dash' in h:        cmd = 'tr - _'
    elif 'curly' in h and 'paren' in h:            cmd = "tr '()' '{}'"
    elif 'square' in h and 'paren' in h:           cmd = "tr '()' '[]'"
    elif 'swap' in h and 'case' in h:              cmd = 'tr A-Za-z a-zA-Z'
    else:
        print(io.recvall(timeout=5).decode()); break
    io.sendline(cmd.encode())

print(io.recvall(timeout=5).decode())
```

## Key takeaways

- `tr a b` swaps in *both* directions — to undo "underscores → dashes" you literally run `tr - _` on the transformed text.
- `rev` is its own inverse, and ROT13 is its own inverse, so the same command applies whether you encoded or decoded.
- Base64 strips trailing newlines cleanly with `base64 -d`; no `--decode` flag wrangling needed.
- Pattern-matching hints with `pwntools` is faster (and less error-prone) than typing under pressure.
