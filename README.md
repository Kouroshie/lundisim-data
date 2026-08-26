# LUNDIsim data

Corner-point reservoir model used for upscaling work.

## File

`LUNDIsim-Fluvial-res0-mesh.grdecl` (75 MB, plain ASCII, written from MATLAB via MRST)

Direct fetch:

```
https://raw.githubusercontent.com/Kouroshie/lundisim-data/main/LUNDIsim-Fluvial-res0-mesh.grdecl
```

## Grid

`SPECGRID` = `128 128 32 1 F`, so NX=128, NY=128, NZ=32 and 524,288 cells.

Keywords present, in file order:

| Keyword   | Meaning |
|-----------|---------|
| `SPECGRID`| grid dimensions |
| `COORD`   | corner-point pillar coordinates |
| `ZCORN`   | corner depths |
| `ACTNUM`  | active-cell flag |
| `PORO`    | porosity |
| `PERMX`   | permeability, x |

`COORD` and `ZCORN` account for nearly all of the file size. Only `PORO` and
`PERMX` are needed for the property panels.

## Parsing

Values are whitespace-delimited, each section ends with `/`, and `--` starts a
comment. Run-length tokens of the form `N*VALUE` expand to `N` copies of
`VALUE`. Cell ordering is x fastest, then y, then z, so a flat array reshapes
as `(NZ, NY, NX)` = `(32, 128, 128)`.

```python
import re, numpy as np

PATH = "LUNDIsim-Fluvial-res0-mesh.grdecl"
NZ, NY, NX = 32, 128, 128
N = NZ * NY * NX

def section(text, kw):
    m = re.search(r'^\s*' + kw + r'\s*$', text, re.M)
    if not m:
        return None
    body = text[m.end(): text.index('/', m.end())]
    out = []
    for tok in body.split():
        if '*' in tok:                      # grdecl run-length form  N*VALUE
            n, v = tok.split('*', 1)
            out.extend([float(v)] * int(n))
        else:
            out.append(float(tok))
    return np.asarray(out, dtype=np.float32)

txt = open(PATH, errors='replace').read()
txt = '\n'.join(l.split('--')[0] for l in txt.split('\n'))   # strip comments

data = {}
for kw in ('PERMX', 'PORO', 'ACTNUM'):
    a = section(txt, kw)
    if a is not None and a.size == N:
        data[kw.lower()] = a.reshape(NZ, NY, NX)
```
