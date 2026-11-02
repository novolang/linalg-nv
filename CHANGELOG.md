# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-10-11

Every body lands.  0.0.3 was the interface; this is the release that
makes it work, and no signature changed.

- **The five decompositions, implemented as published.**  `lalu.factor`
  is Gaussian elimination with partial pivoting, and `LaSingular` means
  a column with no nonzero pivot at all — not a nearly singular one.
  `laqr.factor` is Householder, with the reflector's sign taken from the
  far end so `beta` is a sum of magnitudes rather than their difference.
  `lachol.factor` reads the lower triangle only.  `laeig.factor` and
  `lasvd.factor` are cyclic and one-sided Jacobi under the published
  30-sweep bound.  Every function named in the README's LAPACK table is
  the routine that table names.
- **The refusals are the published ones, to the sentence.**  Each shape
  check names the entry point that looked, `LaSingular` carries the
  column and the pivot, `LaNotPositiveDefinite` carries the position and
  the value, and `LaNoConvergence` carries the sweep count and the
  residual.
- **A second suite, `tests/linalg_edge_tests.nv`, 23 tests.**  The
  oracle's 26 tests are unchanged.  The supplement holds what a success
  path cannot pin: every `LaFault` variant's message, an exactly
  singular matrix (a repeated row) and one an ulp from singular, a
  degenerate spectrum whose zero eigenvalue makes the condition number
  an infinity, the shapes each entry point refuses, a factor assembled
  by hand with a zero on `U`'s diagonal, and the reconstruction undoing
  a pivot that partial pivoting actually made.
- **`lanorm.array_of` deleted.**  A private helper with no caller, and
  the one instrumented line neither suite could reach.
- **Coverage is complete**: 1569 of 1569 instrumented lines under `src/`
  are covered by the two suites, so the release carries no `cov: skip`
  marker.

### Changed

- `novo.toml`: `version` 0.1.0, `stability = "experimental"`, the
  `novo` floor to `>= 0.20.2` (the compiler this was verified against),
  and the `description` without its interface-only tail.
- `README.md`: the `NOT IMPLEMENTED` block and the implementation-status
  table are gone, with nothing in their place; the sentences that said
  the tests fail at run are gone; the Tests section names both suites.
- Each `src/*.nv` header drops the "every body is a `todo()`" sentence.
  The design prose is unchanged.
- `tests/linalg_tests.nv`: the release-stamp sentence in the header is
  gone.  No assertion changed.
- `lanorm.nv`: `array_of` removed (see above); nothing called it.

### Known

- **The one-call forms delegate their shape checks.**  A refusal from
  `lasolve.solve` still names `lalu.factor` or `lalu.solve`, not the
  entry point the caller wrote, because the one-shot form is the two
  lines it is documented to be.  The edge suite pins the wording.
- **`lanorm.condition_estimate` is a lower bound and can undershoot.**
  Hager's iteration stops at the first column that fails to beat the
  best already seen, which is what makes the estimate honest rather
  than an overshoot.  On `[[-3,-1,0],[2,1,-1],[0,1,0]]` it answers 5
  where the true 1-norm condition number is 25/3 ≈ 8.33.  Pinned in the
  edge suite; the doc block says why.
- **One design pin was unpinned until this release's edge suite.**  A
  mutation that stops `lalu.reconstruct` undoing the permutation
  survived both suites before `test_the_reconstruction_undoes_the_pivots_the_factorisation_made`
  was added; the oracle checks the reconstruction against the identity
  only, which swaps nothing.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- ndarray-nv: `^0.0.1` to `^0.1.4`.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Eight modules.  `lalu`, `laqr`, `lachol`, `laeig` and `lasvd` are the
  five decompositions; `lanorm` is the norms and the two condition
  numbers; `lasolve` is the one-call conveniences; `lafault` is every
  refusal.
- **A decomposition is a value that holds its factors**, and that is
  the load-bearing interface.  Every module has the same three-part
  shape: a struct, a `factor` that builds it, and functions that read
  it.  The cubic work and the quadratic work are separated visibly, the
  interesting refusals happen once at a call the caller can see, one
  factorisation answers many questions, and nothing is carried between
  calls — which is what keeps the package at `core`.
- **`lasolve` is a module and not a set of overloads**, because the
  convenience and the value are different promises: a reader who opens
  `lalu` sees a factorisation worth keeping, and one who opens
  `lasolve` sees ten functions that each throw one away.
- **Every function names the LAPACK routine it mirrors.**  OpenBLAS is
  an implementation of LAPACK, so "the reference" is a named routine —
  `dgetrf`, `dgeqrf`, `dpotrf`, `dsyevj`, `dgesvj` — and the README has
  the table.  A result that disagrees with LAPACK is this package's
  fault, and a reader has to know what to compare against.
- **Jacobi for both iterative decompositions, argued.**  Cyclic Jacobi
  for the symmetric eigenproblem and one-sided Jacobi for the SVD: both
  are slower than the QR-shift and Golub-Kahan paths by a constant
  factor, both compute small values to high RELATIVE accuracy on graded
  and badly scaled matrices, and both are simple enough for a reader to
  check.  The faster paths can go behind the same signatures later,
  because `LaEigen` and `LaSvd` say nothing about how they were found.
- **Householder QR and no normal equations.**  Classical Gram-Schmidt
  loses orthogonality by the condition number SQUARED; Householder loses
  it by machine epsilon.  There is no `normal_equations` function
  either: `AᵀA` squares the condition number, and offering it beside
  the QR path would be a footgun with a shorter name.
- **Two least-squares paths, and the difference is stated.**
  `laqr.lstsq` is faster and REFUSES a rank-deficient matrix;
  `lasvd.lstsq` answers the minimum-norm solution.  The README has the
  table.
- **Three conventions that disagree with something**, each stated
  rather than discovered: eigenvalues and singular values sort
  DESCENDING (LAPACK and numpy sort eigenvalues ascending); `lasvd`
  answers `V` and not `Vᵀ`; symmetry is not checked, because
  `lachol` and `laeig` read only the lower triangle and
  `lanorm.symmetry_defect` is where a caller who is unsure asks.
- **Three kinds of refusal, told apart by a predicate.**  A shape fault
  means the calling code is wrong, a singular or indefinite fault means
  the data says no and a fallback is wanted, and a convergence fault
  means raising the sweep bound is reasonable.  `LaSingular` and
  `LaNotPositiveDefinite` carry the value that decided them, because a
  covariance matrix rounded to -1e-18 is a different problem from one
  that is genuinely indefinite.
- **No `@tier(embedded)` claim and no probe.**  Every matrix is an
  `NdFloat`, which is a heap value, and the fixed-size 3x3 and 4x4 case
  already has a home in geometry-nv, which the plan's row says not to
  duplicate.
- Every vector is a matrix factored by hand, an identity that holds for
  every matrix, or a published rule.  Nothing is a transcription of
  LAPACK's output for a random matrix, because a decomposition is not
  unique and such a test would assert against one of many correct
  answers.
