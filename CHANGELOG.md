# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
