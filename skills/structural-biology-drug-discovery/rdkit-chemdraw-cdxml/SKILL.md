---
name: "rdkit-chemdraw-cdxml"
description: "Read, write, and edit ChemDraw CDX/CDXML files with RDKit's rdkit.Chem.rdChemDraw module plus direct XML editing. Parse molecules and reactions from .cdxml/.cdx, write clean molecule structures with good 2D depiction (CoordGen, template alignment), and hand-build or modify the parts RDKit cannot write: reaction arrows, plus signs, reaction schemes/steps, and free text/labels via the CDXML XML schema. Use when generating publication-ready chemical drawings, building reaction schemes programmatically, or modifying existing ChemDraw files. Critical: RDKit writes structures only; round-tripping a reaction through a Mol silently drops arrows and text — this skill shows the XML layer needed to preserve them. For pure molecular analysis (descriptors, fingerprints, SMARTS) use rdkit-cheminformatics; for multi-format 3D conversion use openbabel."
license: "BSD-3-Clause"
---

# RDKit ChemDraw / CDXML Toolkit

## Overview

ChemDraw's native formats are CDX (binary) and CDXML (an XML serialization of the same object tree). RDKit 2022.09+ ships an optional Revvity ChemDraw parser exposed through `rdkit.Chem.rdChemDraw`, which reads molecules **and** reactions from both formats and writes molecule structures. This skill combines that RDKit I/O with direct CDXML XML editing, because RDKit writes only atoms/bonds/coordinates — reaction **arrows**, **plus signs**, **reaction schemes**, and free **text/labels** are ChemDraw canvas objects that must be built or edited at the XML level. The result is a full read → depict → annotate → write → modify workflow for programmatic ChemDraw drawings.

## When to Use

- Convert SMILES/SDF/Mol structures into `.cdxml` files that open cleanly in ChemDraw
- Extract molecules or reactions (reactants/agents/products) from `.cdxml` or `.cdx` files
- Build a reaction scheme programmatically: fragments + reaction arrow + `+` separators + conditions text
- Add captions, atom labels, or annotations to a chemical drawing at specific canvas positions
- Modify an existing ChemDraw file (relabel a group, add a note, reposition an arrow) without losing its arrows/text
- Improve the 2D layout of a structure before export (CoordGen, template alignment, straightening)
- Render a `.cdxml`/`.cdx` to PNG or SVG for a visual quality check of the drawing you generated
- Batch-generate ChemDraw figures for a reaction dataset or SAR table
- Use `rdkit-cheminformatics` instead when you only need descriptors, fingerprints, similarity, or SMARTS — no ChemDraw I/O
- For multi-format 3D structure conversion (MOL2, XYZ, PDB), use `openbabel` instead; this toolkit is 2D ChemDraw-specific

## Prerequisites

- **Python packages**: `rdkit` (2023.03+ recommended; must be built with ChemDraw support), `lxml` (optional, for pretty-printing/XPath); `xml.etree.ElementTree` from the stdlib is sufficient for editing. Optional: `epam.indigo` for rendering CDXML to PNG/SVG (Module 9)
- **Data requirements**: SMILES/Mol objects for writing; `.cdxml` (UTF-8 text) or `.cdx` (binary) files for reading
- **Environment**: Python 3.9+; verify ChemDraw write support at runtime (see below) — some conda builds omit it

> **Check before installing.** In a managed env (pixi/conda), RDKit is likely already present — run `python -c "import rdkit; print(rdkit.__version__)"` first and skip the install if it succeeds. Inside a pixi project, run scripts with `pixi run python ...`.

```bash
# Only if RDKit is not already available:
pip install rdkit lxml
# Then confirm the optional ChemDraw parser is compiled in:
python -c "from rdkit import Chem; print('ChemDraw support:', Chem.HasChemDrawCDXSupport())"
```

## Quick Start

```python
from rdkit import Chem
from rdkit.Chem import rdChemDraw, rdDepictor

mol = Chem.MolFromSmiles("CC(=O)Oc1ccccc1C(=O)O")  # aspirin
rdDepictor.SetPreferCoordGen(True)                  # nicer layouts
rdDepictor.Compute2DCoords(mol)                     # coordinates are REQUIRED before writing

cdxml = rdChemDraw.MolToChemDrawBlock(mol, rdChemDraw.CDXFormat.CDXML)  # -> str
with open("aspirin.cdxml", "w", encoding="utf-8") as fh:
    fh.write(cdxml)
print(f"Wrote {len(cdxml)} chars of CDXML")
```

## Core API

### Module 1: Reading molecules

`MolsFromChemDrawFile` / `MolsFromChemDrawBlock` handle **both** `.cdx` and `.cdxml` and return a tuple of `Mol` objects (one per fragment on the page).

```python
from rdkit import Chem
from rdkit.Chem import rdChemDraw

# From a file (.cdxml or .cdx — format auto-detected)
mols = rdChemDraw.MolsFromChemDrawFile("drawing.cdxml", sanitize=True, removeHs=True)
print(f"Parsed {len(mols)} molecule(s)")
for m in mols:
    print("  ", Chem.MolToSmiles(m))

# From an in-memory block (str for CDXML, bytes for CDX)
block = open("drawing.cdxml", encoding="utf-8").read()
mols = rdChemDraw.MolsFromChemDrawBlock(block, sanitize=True, removeHs=True)

# Legacy CDXML-only parser (no ChemDraw SDK needed) as a fallback:
mols_legacy = Chem.MolsFromCDXML(block, sanitize=True, removeHs=True)
```

### Module 2: Reading reactions (arrows → reactant/product split)

Arrows in ChemDraw carry the reaction semantics. `ReactionsFromChemDrawBlock` interprets `<step>`/`<arrow>` objects and returns `ChemicalReaction`s with reactants, agents, and products split out.

```python
from rdkit.Chem import rdChemDraw, rdChemReactions

block = open("reaction.cdxml", encoding="utf-8").read()

# Note: reaction reader defaults sanitize=False, removeHs=False
rxns = rdChemDraw.ReactionsFromChemDrawBlock(block, sanitize=True, removeHs=False)
for rxn in rxns:
    print("reactants:", [Chem.MolToSmiles(m) for m in rxn.GetReactants()])
    print("agents   :", [Chem.MolToSmiles(m) for m in rxn.GetAgents()])
    print("products :", [Chem.MolToSmiles(m) for m in rxn.GetProducts()])

# CDXML-only equivalent (legacy parser):
rxns2 = rdChemReactions.ReactionsFromCDXMLBlock(block, sanitize=True, removeHs=False)
```

### Module 3: Writing molecule structures

`MolToChemDrawBlock` writes one molecule. **CDXML returns `str`; CDX write is unreliable in `rdChemDraw`** (raises `UnicodeDecodeError`), so use the legacy writer for binary CDX bytes.

```python
from rdkit import Chem
from rdkit.Chem import rdChemDraw, rdDepictor
from rdkit.Chem import rdmolfiles

mol = Chem.MolFromSmiles("c1ccccc1O")
rdDepictor.Compute2DCoords(mol)                      # always compute coords first

# CDXML (text) — preferred, works in both writers
cdxml = rdChemDraw.MolToChemDrawBlock(mol, rdChemDraw.CDXFormat.CDXML)   # str

# CDX (binary) — use the LEGACY writer; rdChemDraw's CDX path is broken
cdx_bytes = Chem.MolToCDXMLBlock(mol, rdmolfiles.CDXMLFormat.CDX)        # bytes
with open("phenol.cdx", "wb") as fh:
    fh.write(cdx_bytes)

if not Chem.HasChemDrawCDXSupport():
    raise RuntimeError("This RDKit build lacks ChemDraw write support")
```

### Module 4: Good molecular depiction

Layout quality is set *before* you write. CoordGen gives more natural, less-overlapping 2D coordinates than the default algorithm; template alignment keeps a common scaffold oriented consistently across a series.

```python
from rdkit import Chem
from rdkit.Chem import rdDepictor
from rdkit.Chem import AllChem

# 1) Prefer CoordGen globally for publication-like layouts
rdDepictor.SetPreferCoordGen(True)

mol = Chem.MolFromSmiles("O=C(Nc1ccc(cc1)S(=O)(=O)N)C")
rdDepictor.Compute2DCoords(mol)

# 2) Clean up: straighten rings/bonds and normalize bond length to a target
rdDepictor.StraightenDepiction(mol)
rdDepictor.NormalizeDepiction(mol)      # scales so the median bond length is uniform
```

```python
# 3) Align a series to a shared scaffold so the core is drawn identically each time
template = Chem.MolFromSmiles("c1ccc(cc1)S(=O)(=O)N")  # sulfonamide core
rdDepictor.Compute2DCoords(template)

series = [Chem.MolFromSmiles(s) for s in
          ["Cc1ccc(cc1)S(=O)(=O)N", "Clc1ccc(cc1)S(=O)(=O)N"]]
for m in series:
    rdDepictor.GenerateDepictionMatching2DStructure(m, template)  # core aligned
    print(Chem.MolToSmiles(m), "aligned to template")
```

### Module 5: Drawing arrows

RDKit cannot write arrows, so build them in CDXML. A reaction arrow is an `<arrow>` object with `Head3D`/`Tail3D` positions (`"x y z"`, y increases downward). The `ArrowheadHead`/`ArrowheadType` attributes control the head; `HeadSize` is in CDXML units (~1000/unit typical).

```python
import xml.etree.ElementTree as ET

def make_arrow(arrow_id, tail_xy, head_xy, kind="Full"):
    """Straight reaction arrow. kind: 'Full' (reaction), 'None' (line)."""
    tx, ty = tail_xy
    hx, hy = head_xy
    arrow = ET.Element("arrow", {
        "id": str(arrow_id),
        "BoundingBox": f"{min(tx,hx)} {min(ty,hy)-4} {max(tx,hx)} {max(ty,hy)+4}",
        "FillType": "None",
        "ArrowheadType": "Solid",
        "ArrowheadHead": kind,          # 'Full', 'HalfLeft', 'HalfRight', 'None'
        "HeadSize": "2250",
        "Head3D": f"{hx} {hy} 0",
        "Tail3D": f"{tx} {ty} 0",
    })
    return arrow

print(ET.tostring(make_arrow(40, (160, 100), (210, 100)), encoding="unicode"))
# Equilibrium/resonance/retrosynthetic arrows: use a <graphic> Line with ArrowType
equil = ET.Element("graphic", {"id": "41", "GraphicType": "Line",
                               "ArrowType": "Equilibrium", "HeadSize": "2250",
                               "BoundingBox": "160 100 210 100"})
```

### Module 6: Reaction schemes and steps

A `<scheme>` groups one or more `<step>` objects. Each `<step>` references other objects **by id**: `ReactionStepReactants`, `ReactionStepProducts`, `ReactionStepArrows`, `ReactionStepPlusses`, and objects above/below the arrow (for conditions). Fragments and the arrow live on the `<page>`; the step just wires their ids together.

```python
import xml.etree.ElementTree as ET

# Plus sign between two reactants (a Symbol graphic)
def make_plus(gid, x, y):
    return ET.Element("graphic", {"id": str(gid), "GraphicType": "Symbol",
                                  "SymbolType": "Plus",
                                  "BoundingBox": f"{x} {y-7} {x+15} {y+8}"})

# Wire reactant/product/arrow ids into one reaction step
step = ET.Element("step", {
    "id": "61",
    "ReactionStepReactants": "10 20",     # fragment ids, space-separated
    "ReactionStepProducts": "50",
    "ReactionStepArrows": "40",
    "ReactionStepPlusses": "30",
    "ReactionStepObjectsAboveArrow": "70",  # e.g. a text id for conditions
})
scheme = ET.Element("scheme", {"id": "60"})
scheme.append(step)
print(ET.tostring(scheme, encoding="unicode"))
```

### Module 7: Adding text and labels

Free text is a `<t>` object positioned by `p="x y"`, containing one or more `<s>` styled-string children. Each `<s>` references a `font` id (from `<fonttable>`) and a `color` index (from `<colortable>`); `size` is in points and `face` is a style bitmask (1=bold, 2=italic, 4=underline; e.g. `96` = subscript/superscript flags used by ChemDraw).

```python
import xml.etree.ElementTree as ET

def make_fonttable(font_id=21, name="Helvetica"):
    ft = ET.Element("fonttable")
    ET.SubElement(ft, "font", {"id": str(font_id), "charset": "x-mac-roman", "name": name})
    return ft

def make_colortable():
    ct = ET.Element("colortable")
    for r, g, b in [(1, 1, 1), (0, 0, 0)]:   # index 0=white(bg), then black...
        ET.SubElement(ct, "color", {"r": str(r), "g": str(g), "b": str(b)})
    return ct

def make_text(tid, x, y, text, font_id=21, size=10, color=0, face=0):
    t = ET.Element("t", {"id": str(tid), "p": f"{x} {y}"})
    s = ET.SubElement(t, "s", {"font": str(font_id), "size": str(size),
                               "color": str(color), "face": str(face)})
    s.text = text
    return t

print(ET.tostring(make_text(70, 175, 92, "reflux, 2 h"), encoding="unicode"))
```

Chemical formulas need **subscripts/superscripts**, which ChemDraw renders by
splitting a `<t>` into multiple `<s>` runs with different `face` bitmasks
(`32`=subscript, `64`=superscript). One run per style change:

```python
import xml.etree.ElementTree as ET

def make_formula(tid, x, y, runs, font_id=3, size=8.5):
    """runs: list of (text, face). face 0=normal, 32=subscript, 64=superscript."""
    t = ET.Element("t", {"id": str(tid), "p": f"{x} {y}"})
    for text, face in runs:
        s = ET.SubElement(t, "s", {"font": str(font_id), "size": str(size),
                                   "color": "0", "face": str(face)})
        s.text = text
    return t

# "Br2 (excess)"  ->  Br<sub>2</sub> (excess)
print(ET.tostring(make_formula(80, 158, 135,
      [("Br", 0), ("2", 32), (" (excess)", 0)]), encoding="unicode"))
# "CO2H" -> CO<sub>2</sub>H
print(ET.tostring(make_formula(81, 690, 730,
      [("-CO", 0), ("2", 32), ("H", 0)]), encoding="unicode"))
```

### Module 8: Editing an existing CDXML file

Because CDXML is generic XML, `ElementTree` round-trips arrows, text, and graphics it does not "understand" — so you can load a real ChemDraw file, tweak it, and save without losing objects. (RDKit's Mol round-trip would drop them.)

```python
import xml.etree.ElementTree as ET

tree = ET.parse("reaction.cdxml")          # DOCTYPE is dropped on re-save (harmless)
root = tree.getroot()
page = root.find("page")

# Relabel every "Cl" text object to "Br"
for t in root.iter("t"):
    for s in t.iter("s"):
        if s.text == "Cl":
            s.text = "Br"

# Append a caption; reuse the file's existing font/color tables
cap = ET.SubElement(page, "t", {"p": "100 300"})
ET.SubElement(cap, "s", {"font": "21", "size": "12", "color": "0"}).text = "Scheme 1"

tree.write("reaction_edited.cdxml", encoding="unicode", xml_declaration=True)
print("Saved reaction_edited.cdxml")
```

### Module 9: Rendering CDXML to PNG (paired deliverable + visual QA)

CDXML is not human-viewable on its own, so **a `.cdxml` should never be handed
over alone — always deliver it together with a rendered `.png` (or `.svg`) of the
same drawing.** The image serves two purposes: it is what the user actually looks
at, and it is your own fastest check for overlapping structures, stray arrows
(from duplicate/degenerate arrow ids), and text colliding with atoms. Render the
PNG next to the CDXML (same basename), verify it visually, then provide both files.
[Indigo](https://lifescience.opensource.epam.com/indigo/) (`epam.indigo`, renderer
bundled) loads a CDXML — a scheme with arrows as a *reaction*, a lone structure as
a *molecule* — and rasterizes arrows, text, and layout faithfully.

```python
from indigo import Indigo
from indigo.renderer import IndigoRenderer

def render_cdxml(cdxml_path, png_path, width=1600):
    ind = Indigo()
    rnd = IndigoRenderer(ind)
    ind.setOption("render-output-format", "png")   # or "svg"
    ind.setOption("render-background-color", "1,1,1")
    ind.setOption("render-image-width", width)
    cdxml = open(cdxml_path, encoding="utf-8").read()
    try:
        obj = ind.loadReaction(cdxml)   # scheme with arrows
    except Exception:
        obj = ind.loadMolecule(cdxml)   # single structure
    rnd.renderToFile(obj, png_path)
    return png_path

# Render the PNG as a sibling of the CDXML, then deliver BOTH files together
from pathlib import Path
src = "scheme.cdxml"
png = str(Path(src).with_suffix(".png"))
render_cdxml(src, png)
print(f"Deliverables: {src} + {png}")
```

RDKit can also draw a *parsed* reaction (`Draw.ReactionToImage(rxn)`), but it
re-lays-out the molecules and drops the original ChemDraw arrows/text/positions —
use Indigo when you want the file rendered as authored.

## Key Concepts

### CDXML coordinate system

Positions use ChemDraw units where **y increases downward** (origin top-left). Atom positions are `p="x y"`; arrows use `Head3D`/`Tail3D="x y z"`; graphics and text carry a `BoundingBox="x1 y1 x2 y2"`. The default bond length is ~30 units. RDKit sometimes emits `<CDXML BondLength="">` (empty) — set a numeric `BondLength` (e.g. `"30"`) on the root when assembling files so ChemDraw scales sanely.

```python
# Shift a fragment's atoms by (dx, dy) to position it on the canvas
def shift_fragment(frag, dx, dy):
    for n in frag.iter("n"):
        x, y = map(float, n.get("p").split())
        n.set("p", f"{x+dx} {y+dy}")
    return frag
```

### Object-id reference model

Every object has a unique integer `id`. Reactions, groups, and brackets refer to their members **by id** rather than by nesting. When you merge fragments from separate RDKit outputs into one page, **renumber all ids to keep them globally unique**, then wire the reaction `<step>` to the new ids.

### RDKit vs XML capability boundary

| Task | RDKit `rdChemDraw` | Direct XML |
|------|--------------------|-----------|
| Read molecules | ✅ | — |
| Read reactions (arrows) | ✅ | — |
| Write molecule structure | ✅ (CDXML) | — |
| Write arrows / plus / scheme | ❌ | ✅ |
| Write text / captions | ❌ | ✅ |
| Preserve objects while editing | ❌ (drops on Mol round-trip) | ✅ |

## Common Workflows

### Workflow 1: SMILES → cleanly depicted single-molecule CDXML

**Goal**: Produce a ChemDraw file for one structure with a good 2D layout.

```python
from rdkit import Chem
from rdkit.Chem import rdChemDraw, rdDepictor

def smiles_to_cdxml(smiles, path):
    mol = Chem.MolFromSmiles(smiles)
    if mol is None:
        raise ValueError(f"Invalid SMILES: {smiles}")
    rdDepictor.SetPreferCoordGen(True)
    rdDepictor.Compute2DCoords(mol)
    rdDepictor.StraightenDepiction(mol)
    cdxml = rdChemDraw.MolToChemDrawBlock(mol, rdChemDraw.CDXFormat.CDXML)
    with open(path, "w", encoding="utf-8") as fh:
        fh.write(cdxml)
    return path

print("Wrote", smiles_to_cdxml("CC(=O)Oc1ccccc1C(=O)O", "aspirin.cdxml"))
```

### Workflow 2: Build a full reaction scheme CDXML (arrow + plus + conditions)

**Goal**: Assemble two reactants, a product, a reaction arrow, a `+` separator, and a text condition into one valid CDXML that ChemDraw opens and RDKit re-parses as a reaction.

```python
from rdkit import Chem
from rdkit.Chem import rdChemDraw, rdDepictor, rdChemReactions
import xml.etree.ElementTree as ET

def _fragment_of(smiles, base_id, dx):
    """Write one molecule to CDXML, extract its <fragment>, renumber ids, shift x."""
    m = Chem.MolFromSmiles(smiles); rdDepictor.Compute2DCoords(m)
    frag = ET.fromstring(rdChemDraw.MolToChemDrawBlock(m)).find("page/fragment")
    remap = {}
    for i, el in enumerate([frag, *frag.iter("n"), *frag.iter("b")]):
        old = el.get("id"); remap[old] = str(base_id + i); el.set("id", remap[old])
    for b in frag.iter("b"):                 # fix bond endpoint references
        b.set("B", remap[b.get("B")]); b.set("E", remap[b.get("E")])
    for n in frag.iter("n"):                 # shift onto the canvas
        x, y = map(float, n.get("p").split()); n.set("p", f"{x+dx} {y}")
    return frag

root = ET.Element("CDXML", {"BondLength": "30"})
page = ET.SubElement(root, "page")

r1 = _fragment_of("CCO", 100, 0);      page.append(r1)      # reactant 1
plus = ET.SubElement(page, "graphic", {"id": "30", "GraphicType": "Symbol",
                                       "SymbolType": "Plus", "BoundingBox": "60 -7 75 8"})
r2 = _fragment_of("CC(=O)O", 200, 120); page.append(r2)     # reactant 2
arrow = ET.SubElement(page, "arrow", {"id": "40", "FillType": "None",
        "ArrowheadHead": "Full", "ArrowheadType": "Solid", "HeadSize": "2250",
        "BoundingBox": "260 0 320 7", "Head3D": "320 3 0", "Tail3D": "260 3 0"})
cond = ET.SubElement(page, "t", {"id": "70", "p": "270 -12"})
ET.SubElement(cond, "s", {"font": "21", "size": "9", "color": "0"}).text = "H+, reflux"
prod = _fragment_of("CCOC(C)=O", 300, 420); page.append(prod)  # product (clear of arrow)

scheme = ET.SubElement(page, "scheme", {"id": "60"})
ET.SubElement(scheme, "step", {"id": "61", "ReactionStepReactants": "100 200",
        "ReactionStepProducts": "300", "ReactionStepArrows": "40",
        "ReactionStepPlusses": "30", "ReactionStepObjectsAboveArrow": "70"})
ft = ET.SubElement(root, "fonttable")
ET.SubElement(ft, "font", {"id": "21", "charset": "x-mac-roman", "name": "Helvetica"})

cdxml = ET.tostring(root, encoding="unicode")
open("esterification.cdxml", "w", encoding="utf-8").write(cdxml)

# Verify: RDKit should re-parse it as a reaction
rxns = rdChemReactions.ReactionsFromCDXMLBlock(cdxml, sanitize=True)
print("Re-parsed reactions:", len(rxns))

# Deliverable pair: render a PNG next to the CDXML (see Module 9) and hand over both
from indigo import Indigo
from indigo.renderer import IndigoRenderer
ind = Indigo(); rnd = IndigoRenderer(ind)
ind.setOption("render-output-format", "png"); ind.setOption("render-image-width", 1600)
rnd.renderToFile(ind.loadReaction(cdxml), "esterification.png")
print("Deliverables: esterification.cdxml + esterification.png")
```

### Workflow 3: Modify an existing ChemDraw file, preserving arrows and text

**Goal**: Open a reaction someone drew, add an annotation and rescale, and save — without the arrow/text loss an RDKit Mol round-trip would cause.

```python
import xml.etree.ElementTree as ET

tree = ET.parse("input_reaction.cdxml")
root = tree.getroot()
page = root.find("page")

# Add a scheme title above the drawing
title = ET.SubElement(page, "t", {"p": "50 -30"})
ET.SubElement(title, "s", {"font": "21", "size": "14", "color": "0", "face": "1"}).text = "Route A"

# Bump every arrow's head size so it reads at print scale
for arrow in root.iter("arrow"):
    arrow.set("HeadSize", "3000")

tree.write("output_reaction.cdxml", encoding="unicode", xml_declaration=True)
print("Edited file saved; arrows and existing text preserved")
```

### Workflow 4: Build a multi-step scheme with the bundled helper (recommended)

**Goal**: Turn a list of `(SMILES, name, conditions)` steps into a laid-out scheme
**and its PNG in one call** — without hand-managing coordinates, object ids, or arrow
placement. This is the reliable path for anything beyond a single reaction: doing it
by hand repeatedly produces duplicate ids, arrows emitted twice, and conditions text
landing on a structure. `scripts/build_reaction_scheme.py` (bundled) does the layout
mechanically — snake grid, unique ids, conditions centered in the clear gap over each
arrow — and always writes the `.cdxml` and `.png` together.

```python
from scripts.build_reaction_scheme import build_scheme  # adjust import path to the skill dir

steps = [
    {"smiles": "O=C1CCCC1", "name": "cyclopentanone"},
    # `conditions` = reagents for the arrow LEADING INTO this step (list = stacked lines)
    {"smiles": "O=C1C(Br)C(Br)C(Br)C1Br", "name": "tetrabromoketone",
     "conditions": ["Br2 (excess)", "AcOH, 25 C"]},
    {"smiles": "O=C1C=CC=C1Br", "name": "2-bromocyclopentadienone",
     "conditions": ["Et2NH", "cold Et2O, -2 HBr"]},
    {"smiles": "O=C1C2C=CC1(Br)C1(Br)C(=O)C=CC21", "name": "endo dimer",
     "conditions": ["spontaneous", "Diels-Alder"]},
    {"smiles": "C12C3C4C1C1C2C3C41", "name": "cubane",
     "conditions": ["(remaining steps...)"]},
]

cdxml_path, png_path = build_scheme(
    steps, "cubane.cdxml", "cubane.png",
    title="Eaton's Total Synthesis of Cubane (1964)", cols=4)
print(f"Deliverables: {cdxml_path} + {png_path}")   # hand over BOTH
```

The helper renders as it builds, so **you can open the PNG, confirm the layout, and
only then deliver both files** — the render step cannot be forgotten because it is
part of the same call.

## Key Parameters

| Parameter | Module / Function | Default | Range / Options | Effect |
|-----------|-------------------|---------|-----------------|--------|
| `format` | `MolToChemDrawBlock` | `CDXFormat.CDXML` | `CDXML`, `CDX` | Output format; use CDXML (str). For CDX bytes use legacy `MolToCDXMLBlock` |
| `sanitize` | `MolsFromChemDrawBlock` | `True` | `True`/`False` | Sanitize parsed mols; set `False` to inspect raw/invalid input |
| `removeHs` | `MolsFromChemDrawBlock` | `True` | `True`/`False` | Convert explicit H to implicit |
| `sanitize` | `ReactionsFromChemDrawBlock` | `False` | `True`/`False` | Reaction reader defaults **False** — pass `True` for clean SMILES |
| CoordGen | `rdDepictor.SetPreferCoordGen` | `False` | `True`/`False` | `True` gives more natural 2D layouts |
| `ArrowheadHead` | `<arrow>` XML | — | `Full`, `HalfLeft`, `HalfRight`, `None` | Arrowhead style on the reaction arrow |
| `ArrowType` | `<graphic>` Line XML | — | `FullHead`, `Equilibrium`, `Resonance`, `RetroSynthetic`, `NoGo` | Special arrow semantics (equilibrium, retrosynthesis, failed) |
| `BondLength` | `<CDXML>` root XML | `""` (RDKit) | numeric, e.g. `30` | Canvas scale; set explicitly to avoid ChemDraw scaling glitches |

## Best Practices

1. **Always compute 2D coordinates before writing.** `MolToChemDrawBlock` serializes existing conformer coordinates; without `Compute2DCoords` you get a degenerate layout.
   ```python
   rdDepictor.SetPreferCoordGen(True); rdDepictor.Compute2DCoords(mol)
   ```

2. **Never round-trip a reaction through an RDKit `Mol`/`ChemicalReaction` if you need to keep the drawing.** Reading loses arrows, plus signs, text, and graphics. To *edit* a reaction file, work on the XML tree (Module 8), not on parsed objects.

3. **Set a numeric `BondLength` on the CDXML root when assembling files by hand.** RDKit emits `BondLength=""`, which can make ChemDraw render structures at the wrong scale relative to arrows/text.

4. **Renumber ids when merging fragments.** Two RDKit outputs both start ids at 1; collisions break reaction-step references. Assign a disjoint id block per fragment, then wire the `<step>` to the new ids.

5. **Prefer CDXML (text) over CDX (binary).** CDXML is diff-able, editable, and reliably written. `rdChemDraw.MolToChemDrawBlock(..., CDX)` currently raises `UnicodeDecodeError`; only the legacy `Chem.MolToCDXMLBlock(mol, CDXMLFormat.CDX)` returns valid CDX bytes.

6. **Verify write support at runtime.** Guard with `Chem.HasChemDrawCDXSupport()` — some conda builds ship without the ChemDraw parser and will raise on write.

7. **Render heteroatom labels from `Element`, never as redundant free text.** A `<n Element="8">` node makes ChemDraw draw "O" automatically. Adding a separate `<t>O</t>` on top of it double-renders the label (a common auto-generation bug). Free `<t>` text is for captions, conditions, and compound names — not atom symbols.

8. **Keep bond length uniform across every fragment.** A scheme where molecules are drawn at bond lengths of 30, 24, and 22 looks broken. RDKit emits a consistent scale per call, but merging fragments from mixed sources can drift — normalize them (`rdDepictor.NormalizeDepiction`) or rescale each fragment's coordinates to a common bond length before assembling. Vary size deliberately (e.g. enlarge a crowded cage), never accidentally.

9. **A reaction scheme needs real arrow objects.** Structures floating on a page with no `<arrow>` (or `<graphic>` line) between them is not a scheme — it is a pile of molecules. Draw one arrow per step and place its conditions text above (arrow y − ~20) and below (arrow y + ~16) the arrow line.

10. **Write a complete document header.** ChemDraw renders most reliably when the `<CDXML>` root carries `BondLength`, `LabelFont`/`LabelSize`, `CaptionFont`/`CaptionSize`, and the `<page>` has explicit `Width`/`Height`/`BoundingBox`, plus a standard `<colortable>`/`<fonttable>`. See `references/cdxml-schema-reference.md` for a copy-paste header.

11. **XML-escape special characters in text.** `<`, `>`, and `&` inside a `<s>` run must be written as `&lt;`, `&gt;`, `&amp;` (e.g. a retrosynthesis note "3 -> 4" becomes "3 -&gt; 4"). `ElementTree` escapes automatically when you set `.text`; only hand-written strings need manual escaping.

12. **Deliver the `.cdxml` and a rendered `.png` together, never the CDXML alone** (Module 9). CDXML is not viewable by eye, so the image is both the user's actual view and your own reliable check — rendering immediately exposes stray lines from a duplicate/degenerate arrow id, structures overlapping an arrow, condition text sitting on top of atoms, and a compressed scheme that dropped intermediates. Render the PNG next to the CDXML, run the lint recipe below (duplicate ids, repeated adjacent fragments, bond-length spread), fix what either surfaces, then hand over both files.

## Common Recipes

### Recipe: Validate a hand-built CDXML by re-reading it

When to use: confirm your assembled arrows/steps are semantically valid.

```python
from rdkit.Chem import rdChemReactions
cdxml = open("esterification.cdxml", encoding="utf-8").read()
rxns = rdChemReactions.ReactionsFromCDXMLBlock(cdxml, sanitize=True)
assert rxns, "No reaction parsed — check ReactionStep* id references"
print("OK:", rxns[0].GetNumReactantTemplates(), "reactants,",
      rxns[0].GetNumProductTemplates(), "products")
```

### Recipe: Batch SMILES → CDXML files

When to use: generate a folder of ChemDraw figures for a compound list.

```python
from rdkit import Chem
from rdkit.Chem import rdChemDraw, rdDepictor
from pathlib import Path

rdDepictor.SetPreferCoordGen(True)
Path("out").mkdir(exist_ok=True)
for i, smi in enumerate(["CCO", "c1ccccc1", "CC(=O)O"]):
    m = Chem.MolFromSmiles(smi); rdDepictor.Compute2DCoords(m)
    Path(f"out/mol_{i}.cdxml").write_text(
        rdChemDraw.MolToChemDrawBlock(m, rdChemDraw.CDXFormat.CDXML), encoding="utf-8")
print("Wrote", len(list(Path('out').glob('*.cdxml'))), "files")
```

### Recipe: Pretty-print CDXML for inspection

When to use: read the object tree while debugging assembly.

```python
import xml.dom.minidom as minidom
pretty = minidom.parseString(open("esterification.cdxml", encoding="utf-8").read())
print(pretty.toprettyxml(indent="  ")[:1500])
```

### Recipe: Lint a generated scheme before rendering

When to use: catch the common auto-generation defects (duplicate ids, an
accidentally repeated structure, uneven scale) that render as stray lines,
floating duplicates, or mismatched sizes.

```python
import xml.etree.ElementTree as ET
from collections import Counter
import statistics

def lint_cdxml(path):
    root = ET.fromstring(open(path, encoding="utf-8").read())
    problems = []
    # 1) duplicate ids (two <arrow id="120">, or a node id reused by a <t>)
    ids = [e.get("id") for e in root.iter() if e.get("id")]
    dups = [i for i, c in Counter(ids).items() if c > 1]
    if dups:
        problems.append(f"duplicate ids: {dups}")
    # 2) bond-length spread across fragments (should be near-uniform)
    def med_bond(fr):
        pos = {n.get("id"): tuple(map(float, n.get("p").split()))
               for n in fr.iter("n") if n.get("p")}
        ds = [((pos[b.get('B')][0]-pos[b.get('E')][0])**2 +
               (pos[b.get('B')][1]-pos[b.get('E')][1])**2)**0.5
              for b in fr.iter("b") if b.get("B") in pos and b.get("E") in pos]
        return statistics.median(ds) if ds else 0
    bl = [round(med_bond(fr), 1) for fr in root.iter("fragment")]
    # Flag only gross scale mismatches; deliberate emphasis (e.g. a larger
    # cage) is fine, so use a loose threshold rather than demanding uniformity.
    if bl and (max(bl) - min(bl)) > 0.35 * max(bl):
        problems.append(f"gross bond-length mismatch across fragments: {bl}")
    # 3) an <arrow> shorter than half a bond (degenerate)
    for a in root.iter("arrow"):
        if a.get("Head3D") and a.get("Tail3D"):
            hx, hy, _ = map(float, a.get("Head3D").split())
            tx, ty, _ = map(float, a.get("Tail3D").split())
            if ((hx-tx)**2 + (hy-ty)**2)**0.5 < 15:
                problems.append(f"degenerate arrow id={a.get('id')} (too short)")
    return problems

issues = lint_cdxml("scheme.cdxml")
print("CLEAN" if not issues else "ISSUES:\n  " + "\n  ".join(issues))
```

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| `RuntimeError`/exception on `MolToChemDrawBlock` | RDKit built without ChemDraw support | Check `Chem.HasChemDrawCDXSupport()`; install a build with the Revvity parser (e.g. recent `pip install rdkit`) |
| `UnicodeDecodeError` writing CDX | `rdChemDraw.MolToChemDrawBlock(..., CDX)` mis-decodes binary | Use legacy `Chem.MolToCDXMLBlock(mol, CDXMLFormat.CDX)` for CDX bytes, or write CDXML instead |
| Structure written but flat/overlapping | No 2D coordinates computed | Call `rdDepictor.Compute2DCoords(mol)` before writing |
| Edited reaction lost its arrows/text | File was round-tripped through `MolsFromChemDrawBlock` → `MolToChemDrawBlock` | Edit the XML tree directly (`ElementTree`); RDKit writes structures only |
| Assembled reaction won't parse back as a reaction | `<step>` references ids that don't exist (id collision after merge) | Renumber all fragment ids to be globally unique, then update `ReactionStep*` attributes |
| Structures and arrows rendered at mismatched sizes | `<CDXML BondLength="">` empty | Set a numeric `BondLength` (e.g. `"30"`) on the root element |
| `mols` tuple empty on read | Wrong format assumed, or unsanitizable structure | Retry with `sanitize=False` to inspect; confirm the file is genuine CDX/CDXML |
| Text shows wrong/no font | `<s font>` references a missing `<fonttable>` id | Include a `<fonttable>` with the referenced `font id`, or reuse the file's existing one |
| Atom labels appear doubled/overlapping ("OO", "BrBr") | Redundant free-text `<t>` labels added on top of `Element` nodes | Remove the free `<t>` atom labels; let `<n Element="…">` render the symbol |
| Molecules in a scheme are different sizes | Fragments assembled at different bond lengths | Rescale each fragment to one bond length (≈30) before assembling; `rdDepictor.NormalizeDepiction` per mol |
| Structures present but no reaction shown | Scheme has no `<arrow>`/`<graphic>` line objects | Add an arrow per step; wire ids in `<step>` if RDKit must re-read it as a reaction |
| Parser error on text with `<`, `>`, `&` | Unescaped special characters in a `<s>` run | Escape as `&lt;` `&gt;` `&amp;` (automatic when using `ElementTree` `.text`) |
| Stray line crosses a structure in the render | Duplicate or degenerate `<arrow>` (e.g. two `<arrow id="120">`, one tiny) | Run the lint recipe; give every arrow a unique id and a real length (Head3D≠Tail3D) |
| Condition text overlaps atoms/structures | Text placed on the structure instead of clear of the arrow | Put conditions above (arrow y − ~15) and below (arrow y + ~15); leave horizontal gaps between fragments |
| Same structure appears twice / floating duplicate | A fragment was copied (e.g. a "from above" continuation) | Remove the duplicate fragment; lint recipe flags repeated adjacent SMILES |
| `render*` options ignored / no image | Indigo renderer not installed, or CDXML loaded as molecule when it has arrows | `pip install epam.indigo`; try `loadReaction` first, fall back to `loadMolecule` |

## Bundled Resources

- `references/cdxml-schema-reference.md` — element/attribute cheat-sheet for `n`, `b`, `arrow`, `graphic`, `step`/`scheme`, `t`/`s`, `fonttable`, `colortable`; coordinate conventions; and the `ArrowType`/`GraphicType`/`SymbolType`/font-face enumerations, distilled from the CambridgeSoft CDX/CDXML specification.
- `scripts/build_reaction_scheme.py` — assemble a multi-step scheme from `(smiles, name, conditions)` steps and render the PNG in one call (Workflow 4). Handles grid layout, globally unique object ids, single arrows, and conditions text placed clear of structures — the defects that recur when schemes are hand-built. Runnable as a library (`build_scheme(...)`) or CLI (`python build_reaction_scheme.py steps.json out.cdxml out.png "Title"`).

## Related Skills

- **rdkit-cheminformatics** — descriptors, fingerprints, similarity, SMARTS; use for analysis once molecules are parsed from CDXML
- **datamol-cheminformatics** — higher-level RDKit wrapper for batch standardization before drawing
- **openbabel** — multi-format 2D/3D conversion (MOL2, XYZ, PDB) when you need formats beyond ChemDraw

## References

- [RDKit `rdkit.Chem.rdChemDraw` documentation](https://www.rdkit.org/docs/source/rdkit.Chem.rdChemDraw.html) — `MolsFromChemDraw*`, `ReactionsFromChemDraw*`, `MolToChemDrawBlock`, `CDXFormat`
- [RDKit `rdMolDraw2D` / depiction docs](https://www.rdkit.org/docs/source/rdkit.Chem.Draw.rdMolDraw2D.html) — 2D coordinate generation and CoordGen
- [CDX/CDXML format specification (CambridgeSoft SDK mirror)](https://chemapps.stolaf.edu/iupac/cdx/sdk/) — object and property reference for arrows, graphics, reactions, and text
- [RDKit CDXML test fixtures](https://github.com/rdkit/rdkit/tree/master/Code/GraphMol/test_data/CDXML) — real ChemDraw-authored `.cdxml` examples
- [Indigo toolkit (`epam.indigo`)](https://lifescience.opensource.epam.com/indigo/) — CDX/CDXML loading and PNG/SVG rendering used in Module 9
