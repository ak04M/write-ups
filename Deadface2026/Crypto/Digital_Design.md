# DEADFACE CTF Write-up — Binary Logic

## Challenge

Investigators found a post by **lilith** on GhostTown that contained a large binary string and an alphabetic string using the letters `A`, `O`, `N`, and `X`.

**Post:** https://ghosttown.deadface.io/t/51/four-letters-still-too-many-for-you

The goal was to decode the message and submit it as `deadface{FLAGTEXT}`.

## 1. Identify the Operations

The four letters represent Boolean logic operations:

- `A` — AND
- `O` — OR
- `N` — NOT
- `X` — XOR

The binary string and operation string must be processed together, from left to right. `N` consumes one input bit, while `A`, `O`, and `X` each consume two input bits. Every operation produces one output bit.

## 2. Decode the Strings

We saved the binary chunks and their corresponding operation strings in `de.py` and used the following Python script:

```python
bits = [
    "001100000", "10110011", "001110101",
    "100111010", "010101000", "101111101",
    "100101100", "00001100", "10001010",
    "100000010", "0100001110"
]

ops = [
    "XXONO", "NXNOA", "OAOON", "NXXAO",
    "NAAXO", "XANXO", "ANOXA", "ANOXN",
    "NONOX", "ANXAX", "AOAAX"
]

def decode(binary, sequence):
    result = []
    pos = 0

    for op in sequence:
        if op == "N":
            result.append(str(1 - int(binary[pos])))
            pos += 1
        else:
            a, b = map(int, binary[pos:pos+2])
            pos += 2

            if op == "A":
                result.append(str(a & b))
            elif op == "O":
                result.append(str(a | b))
            elif op == "X":
                result.append(str(a ^ b))

    return "".join(result)

output = "".join(
    decode(b, op) for b, op in zip(bits, ops)
)

print("Intermediate bits:", output)
print("5-bit groups:", [
    output[i:i+5] for i in range(0, len(output), 5)
])
```

## 3. Intermediate Output

Running the script with:

```bash
python3 de.py
```

produced the following output:

```text
Intermediate bits: 0001001001011100000110010110010110001111001110100100011

5-bit groups: ['00010', '01001', '01110', '00001', '10010',
               '11001', '01100', '01111', '00111', '01001',
               '00011']
```

## 4. Convert the 5-Bit Groups to Letters

The next step was to convert each 5-bit binary group into its decimal equivalent and map the resulting number to a letter using `1 = A`, `2 = B`, and so on, up to `26 = Z`.

| 5-bit group | Decimal | Letter |
|---|---:|---|
| `00010` | 2 | B |
| `01001` | 9 | I |
| `01110` | 14 | N |
| `00001` | 1 | A |
| `10010` | 18 | R |
| `11001` | 25 | Y |
| `01100` | 12 | L |
| `01111` | 15 | O |
| `00111` | 7 | G |
| `01001` | 9 | I |
| `00011` | 3 | C |

Combining the decoded letters gives:

```text
BINARYLOGIC
```

## Flag

The final flag is:

```text
deadface{BINARYLOGIC}
```
