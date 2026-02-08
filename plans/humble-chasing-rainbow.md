# Kakuro Solver - Java 21

## Context
Build a CLI app that reads kakuro puzzles from CSV and solves them via backtracking.

## CSV Format
- `#` = black cell, `_` = empty (to solve), `D\A` = clue (down\across, omit for none)

## Files
```
src/kakuro/
  Cell.java       - sealed interface: Black, Clue, Empty records
  Position.java   - record(row, col)
  Run.java        - record(target, List<Position>)
  Grid.java       - parses cells, extracts runs from clues
  CsvParser.java  - CSV → Grid
  Solver.java     - backtracking + constraint pruning (unique digits, sum bounds)
  Main.java       - CLI entry point, formatted output
puzzles/
  easy.csv        - verified 3x3 example
```

## Solver Strategy
- Order cells most-constrained-first (shortest run)
- Filter candidates by digits already used in adjacent runs
- Prune via: no duplicates, partial sum ≤ target, min/max bounds check on remaining cells

## Build & Run
```bash
javac -d out src/kakuro/*.java
java -cp out kakuro.Main puzzles/easy.csv
```

## Verification
Run with easy.csv, expect solution: 1,2,3,4 in the empty cells.
