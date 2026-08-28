# lundisim-data

Reservoir model and benchmark data, stored as plain git objects (no Git LFS) so
every file is fetchable directly from `raw.githubusercontent.com`.

Base URL for any file below:

```
https://raw.githubusercontent.com/Kouroshie/lundisim-data/main/<path>
```

## Contents

### LUNDIsim corner-point models

| Path | Size | Notes |
|------|------|-------|
| `LUNDIsim-Fluvial-res0-mesh.grdecl` | 78,454,206 B | fluvial realization, geometry plus `ACTNUM`, `PORO`, `PERMX` |
| `lundisim/LUNDIsim-Base-Grid-res0-mesh.grdecl` | 58,006,955 B | same grid, geometry only: `SPECGRID`, `COORD`, `ZCORN`, `ACTNUM` (no properties) |
| `lundisim/LUNDIsim-Fluvial-res0-permeability.jpg` | 228,377 B | rendered permeability field |
| `lundisim/LUNDIsim-Fluvial-res0-porosity.jpg` | 223,427 B | rendered porosity field |
| `lundisim/README.pdf` | 607,427 B | original model documentation |

Both grdecl files carry `SPECGRID = 128 128 32 1 F`, so NX=128, NY=128, NZ=32
and 524,288 cells. Both were written from MATLAB via the MRST toolbox and are
plain ASCII.

The fluvial mesh stays at the repository root rather than under `lundisim/` so
that the previously published raw URL keeps working.

### SPE benchmark cases

| Path | Size | Notes |
|------|------|-------|
| `spe/SPE1CASE1.DATA` | 11,927 B | SPE1, `DIMENS 10 10 3` |
| `spe/SPE1CASE2.DATA` | 11,907 B | SPE1 variant, `DIMENS 10 10 3` |
| `spe/SPE3CASE1.DATA` | 15,558 B | SPE3, `DIMENS 9 9 4` |
| `spe/SPE3CASE2.DATA` | 15,439 B | SPE3 variant, `DIMENS 9 9 4` |
| `spe/SPE9.DATA` | 15,927 B | SPE9, `DIMENS 24 25 15` |
| `spe/PERMVALUES.DATA` | 142,483 B | SPE9 permeability include file |
| `spe/TOPSVALUES.DATA` | 10,996 B | SPE9 `TOPS` include file |
| `spe/SPE10.docx` | 110,669 B | SPE10 model description |
| `spe/spe_perm.dat` | 57,946,647 B | SPE10 permeability, kx then ky then kz |
| `spe/spe_phi.dat` | 19,121,390 B | SPE10 porosity |

The decks are the Statoil/Equinor releases, distributed under the Open Database
License (ODbL 1.0) with contents under the Database Contents License (DbCL 1.0);
the license header is preserved at the top of each `.DATA` file.

`spe_perm.dat` and `spe_phi.dat` are whitespace-delimited scalars, six per line,
for the SPE10 grid of 60 x 220 x 85 = 1,122,000 cells. `spe_phi.dat` holds one
value per cell (187,000 lines). `spe_perm.dat` holds three full fields back to
back, kx, ky, kz, for 3,366,000 values in total. Ordering is x fastest, so:

```python
import numpy as np

phi = np.loadtxt("spe_phi.dat").reshape(85, 220, 60)          # (nz, ny, nx)
k   = np.loadtxt("spe_perm.dat").ravel()
kx, ky, kz = (a.reshape(85, 220, 60) for a in np.split(k, 3))
```

## Byte fidelity

`.gitattributes` sets `* -text`, which disables all CRLF/LF translation. Files
served from `raw.githubusercontent.com` are therefore byte-identical to the
originals, on any platform. Git LFS is deliberately not used: it would make the
raw URLs return a small pointer file instead of the data.

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
