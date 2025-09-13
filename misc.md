
# SoHard — Misc (1000 pts) — Write‑up

**Event:** CyberCon (assumed from flag format)  
**Category:** Misc  
**Flag format:** `cybercon{...}`

---

## Challenge
> _It's so hard:_

```
rrFrrrrroROforrrErFrrroROfoRRRErrrFroRRROfoERRRERRRRRRRRRRRRRERRRRFroRRROfoRERRRRRRRRRRRRErEFrrroROfoRErrrrErFroRRROforErrFrrroROforrERRFroRRROfoEFrrroROforErrrrrrrrrrrrErrrrrrrErrFrrroROforERFroRRROfoERRErrERRRRRRRERRRRRRRRRRRRERRRFroRRROfoRRERRRRRRRRERRRRRErrrrrrrrrrrrrrrERRRRRRRErrrrrrrERRRRERRRRRRRREorrFrroRRROfoE
```

We are given a long string made only from the seven characters `r R f F e E o O`.
It looks like noisy text, but the limited alphabet is a hint that it may be a **codebook / esolang mapping**.

---

## Quick observations
- Alphabet (case‑sensitive): `{r,R,f,F,e,E,o,O}` (7 symbols).  
- Character counts (roughly): many `R`/`r`, fewer `F`/`f` — suggests **increment/decrement** and **loop brackets** like in **Brainf***.
- Brainf*** (BF) uses exactly eight symbols: `+ - < > [ ] . ,`.  
  Here we see **seven**; no comma is needed if the program only **prints** a string.

Heuristic guess:
- `R` ↔ `+`, `r` ↔ `-` (they appear most, like value changes)
- `{O, o}` ↔ `{>, <}` (pointer moves; capital vs lowercase for right/left)
- `{F, f}` ↔ `[{, }]` (balanced brackets)
- `E` ↔ `.` (output)
- `e` is absent in the provided text (not needed).

---

## Reproduce: map → translate → interpret

Below is a minimal Python script that (1) translates the custom alphabet to BF, and (2) runs a tiny BF interpreter to print the output.

```python
s = "rrFrrrrroROforrrErFrrroROfoRRRErrrFroRRROfoERRRERRRRRRRRRRRRRERRRRFroRRROfoRERRRRRRRRRRRRErEFrrroROfoRErrrrErFroRRROforErrFrrroROforrERRFroRRROfoEFrrroROforErrrrrrrrrrrrErrrrrrrErrFrrroROforERFroRRROfoERRErrERRRRRRRERRRRRRRRRRRRERRRFroRRROfoRRERRRRRRRRERRRRRErrrrrrrrrrrrrrrERRRRRRRErrrrrrrERRRRERRRRRRRREorrFrroRRROfoE"

# 1) Mapping (deduced by frequency + bracket balance)
M = {
    'R': '+', 'r': '-',
    'O': '>', 'o': '<',
    'F': '[', 'f': ']',
    'E': '.',  # print
}

bf = ''.join(M[ch] for ch in s)

# 2) Tiny Brainf*** interpreter
def bf_run(code, tape_size=30000):
    tape = [0]*tape_size
    p = 0
    out = []
    i = 0

    # build jump table for [] once
    stack, jmp = [], {}
    for idx, c in enumerate(code):
        if c == '[':
            stack.append(idx)
        elif c == ']':
            j = stack.pop()
            jmp[j] = idx
            jmp[idx] = j

    while i < len(code):
        c = code[i]
        if c == '>': p += 1
        elif c == '<': p -= 1
        elif c == '+': tape[p] = (tape[p] + 1) & 0xFF
        elif c == '-': tape[p] = (tape[p] - 1) & 0xFF
        elif c == '.': out.append(chr(tape[p]))
        elif c == ',': pass  # not used
        elif c == '[' and tape[p] == 0: i = jmp[i]
        elif c == ']' and tape[p] != 0: i = jmp[i]
        i += 1

    return ''.join(out)

print(bf_run(bf))
```

### Output
```
cybercon{was_that_a_frain_f_ck}
```

---

## Why this mapping?
1. **Bracket balance test**: treating `F`/`f` as `[`/`]` creates a valid, perfectly matched loop structure; other pairings fail or create impossible jumps.
2. **Program shape**: after translation, the code looks like typical BF that builds ASCII with loops and pointer moves (`+/-`, `</>`), then prints (`.`).
3. **Minimal alphabet**: no `,` is present — consistent with output‑only BF programs used to print flags.

---

## Flag
**`cybercon{was_that_a_frain_f_ck}`**

---

## Notes & Pitfalls
- Be careful: the character in the string is capital **`O`** (letter O), not digit `0`.
- If you brute‑force all plausible pairings for `[` `]` among `{F,f,O}`, only `F`/`f` works cleanly with balanced loops.
- Any standard BF interpreter will also print the same flag once you apply the mapping above.
