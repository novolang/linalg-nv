# linalg-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Dense linear algebra over ndarray-nv matrices: the five decompositions
that between them answer every question anybody asks about a matrix.

- **LU** with partial pivoting — solve, determinant, log-determinant,
  inverse.
- **QR** by Householder reflections — least squares, residuals,
  orthogonal bases.
- **Cholesky** — the fast path for covariance matrices, Gram matrices
  and Hessians, and the standard test for positive definiteness.
- **Symmetric eigendecomposition** by cyclic Jacobi — spectra, inertia,
  matrix powers, principal components.
- **SVD** by one-sided Jacobi — rank, pseudoinverse, minimum-norm least
  squares, condition number, best low-rank approximation.

Beside them: five matrix norms and four vector ones, the two condition
numbers, and a `lasolve` module of one-call convenience forms for the
caller who has one system and is not coming back.

```
novo pkg add linalg-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use ndfloat
use lalu

// Fifty right-hand sides against one matrix — a Newton iteration, a
// Kalman filter, a simulation step.
fn step_many(a: NdFloat, sides: [NdFloat]) -> Result<[NdFloat], LaFault>
    let lu = lalu.factor(a)!
    sides.map(fn(b) => lalu.solve(lu, b))
```

`factor` is `n³/3` and runs once; each `solve` is `n²`. The same loop
written against a `solve(a, b)` that took a matrix would do the cubic
work fifty times, and nothing in its signature would have said so.

## The layer, and why

`core` — no effects at all.

A decomposition is arithmetic over numbers the caller already holds.
Nothing is read, nothing is written, no clock and no entropy is
consulted, and no scratch state is carried between calls. The budget is
`[]` on every one of the 89 public functions.

**No `@tier(embedded)` claim, and there is no
`tests/embedded_probe.nv`.** The audit's `core-embedded` row passes and
says the package makes no claim, which is the honest outcome. Two
reasons:

- Every matrix here is an `NdFloat`, which is a heap value: a list, a
  shape and an offset. `@tier(embedded)` refuses list literals
  (SPEC § 14.4), so nothing in this package reaches a device as it
  stands.
- **The fixed-size case already has a home.** A device that needs 3×3
  and 4×4 arithmetic wants geometry-nv, which has the 2-D transforms;
  duplicating them here — which the plan's row explicitly says not to
  do — would put two implementations of a rotation matrix on the grid.
  If a fixed-size 3×3/4×4 *decomposition* module is wanted later it is
  a new row and this lane's report names it.

## The load-bearing interface

```novo
pub struct LaLu
pub fn factor(a: NdFloat) -> Result<LaLu, LaFault> []
pub fn solve(lu: LaLu, b: NdFloat) -> Result<NdFloat, LaFault> []
```

**A decomposition is a value that holds its factors**, and every module
here has the same three-part shape: a struct, a `factor` that builds it,
and functions that read it.

That is the whole design, and it is load-bearing for four reasons:

1. **It separates the cubic work from the quadratic work, visibly.**
   `factor` is `n³/3`; `solve` is `n²`. A caller with fifty
   right-hand sides can see which one to hoist out of the loop, because
   the types say so.
2. **It separates the fallible half from the reliable half.** `factor`
   is where a matrix can be singular, indefinite or non-convergent.
   A `solve` against a good factorisation can only fail on the
   right-hand side's own length. So the interesting refusals happen
   once, at a call the caller can see.
3. **One factorisation answers many questions.** An `LaLu` gives a
   solve, a determinant, a log-determinant, an inverse and a condition
   estimate; an `LaSvd` gives rank, pseudoinverse, condition, 2-norm,
   least squares and the best rank-`k` approximation. Every one of
   those over factors already computed.
4. **It keeps the package at `core` with nothing carried between
   calls.** The C libraries make a decomposition a handle with an
   `init` and a `free`; here it is a value that can live in a struct
   beside the matrix it came from, be copied, be used from two places,
   and never be stale, because nothing mutates it.

The price is that a one-shot caller has two calls instead of one, and
that is what `lasolve` is for — a module of two-line conveniences, each
of which says in its own doc comment that it throws the factorisation
away and names the module a repeat caller should use instead. The
convenience and the value are different promises, and putting them in
separate modules is that difference written where a reader will meet it.

## OpenBLAS is the reference, and every function names its routine

The plan's row says OpenBLAS is the numerical reference. OpenBLAS is an
implementation of the **LAPACK** and **BLAS** interfaces, so what each
function here mirrors is a named LAPACK routine — and every doc comment
says which:

| this package | LAPACK | what it is |
| --- | --- | --- |
| `lalu.factor` | `dgetrf` | LU with partial pivoting |
| `lalu.solve` | `dgetrs` | solve from the factors |
| `lalu.inverse` | `dgetri` | inverse from the factors |
| `lanorm.condition_estimate` | `dgecon` | 1-norm condition estimate |
| `laqr.factor` | `dgeqrf` | Householder QR |
| `laqr.q` | `dorgqr` | form `Q` explicitly |
| `laqr.apply_q` | `dormqr` | multiply by `Q` without forming it |
| `laqr.lstsq` | `dgels` | full-rank least squares |
| `lachol.factor` | `dpotrf` | Cholesky |
| `lachol.solve` | `dpotrs` | solve from the factors |
| `lachol.inverse` | `dpotri` | inverse from the factors |
| `laeig.factor` | `dsyevj` | symmetric eigen, Jacobi |
| `lasvd.factor` | `dgesvj` | SVD, one-sided Jacobi |
| `lasvd.lstsq` | `dgelss` | minimum-norm least squares |
| `lanorm.norm` | `dlange` | matrix norms (not the 2-norm) |
| `lanorm.vector_norm` | `dnrm2` | scaled Euclidean norm |
| `lanorm.dot` | `ddot` | dot product |
| `lasolve.solve_lower` / `solve_upper` | `dtrsv` | triangular solve |

That table is not decoration. **A result that disagrees with LAPACK is
this package's fault**, and a reader checking one needs to know what to
compare against — and needs to know that the comparison is against a
*routine*, not against "OpenBLAS in general".

## Two algorithm choices, argued

The plan asked which iterative algorithm each of the two hardest
decompositions uses. Both answers are Jacobi, for the same two reasons,
and both are **reversible behind these signatures**.

**Symmetric eigen: cyclic Jacobi, not tridiagonal QR-shift.**
QR-shift (`dsyev`) is faster — about `(4/3)n³` against Jacobi's ten to
twenty. Jacobi is more *accurate* where it counts: it computes the small
eigenvalues of a graded matrix to high **relative** accuracy, where the
tridiagonal reduction has already lost them to the absolute error of the
largest. A covariance matrix whose eigenvalues span ten orders of
magnitude is exactly the matrix a principal-component analysis is run
on. Jacobi is also far simpler — one rotation in a loop, no deflation,
no shift strategy, no tridiagonal form — which for a first
implementation a reader has to be able to check is worth a constant
factor.

**SVD: one-sided Jacobi, not Golub–Kahan.** The same trade.
Golub–Kahan (`dgesdd`) bidiagonalises and iterates, and is what numpy
calls. One-sided Jacobi (`dgesvj`) orthogonalises pairs of columns until
they are orthogonal, and Demmel and Veselić proved it computes every
singular value to high relative accuracy for a matrix `D B` with `B`
well conditioned — which is any matrix whose columns are in different
units, which is any design matrix. The bidiagonal reduction destroys
that before the iteration begins.

**Both are implementation choices behind a fixed API.** `LaEigen` holds
values and vectors; `LaSvd` holds `U`, `Σ` and `V`. Neither says how it
was found, so putting the faster path in later changes no caller. If a
benchmark says the constant factor matters, that is what happens.

## Accuracy, and what the tests hold

Two tolerances, and the suite uses them deliberately:

- **`1e-12`** on a direct arithmetic identity — a norm of a small
  matrix, a trace, a dot product. These are a handful of flops and
  the double-precision floor is around `1e-16`.
- **`1e-10`** on anything that has been through a factorisation — a
  reconstruction, a solve, an eigenvalue, a singular value. A
  backward-stable decomposition of an `n × n` matrix has an error of
  about `n · eps · ||A||`, which for the sizes in the suite is around
  `1e-14`; four orders of margin absorbs the conditioning of the test
  matrices without admitting anything an algorithmic mistake would
  produce.

The gap between those two numbers and anything a wrong implementation
gives is enormous: a decomposition that is wrong is wrong by O(1), not
by `1e-9`. The suite is built to make that so, by asserting **identities
rather than transcribed outputs**:

- the factors multiply back to the original (`reconstruct`, on all five)
- `Q` is orthogonal (`lanorm.orthogonality_defect`)
- the eigenvalues sum to the trace and multiply to the determinant
- the singular values are non-negative and descending
- Eckart–Young: the rank-`k` truncation of a rank-`k` matrix is exact
- the pseudoinverse of a full-rank square matrix is its inverse

This matters because **a decomposition of an arbitrary matrix is not
unique.** `Q`'s column signs, an eigenvector's direction and a singular
vector's sign are all free, so a test that hard-coded LAPACK's output
for a random matrix would be asserting against one of many correct
answers. The matrices the suite factors by hand — the identity, a
diagonal, a 2×2, the swap matrix `[[0,1],[1,0]]` that needs a pivot on
the first step — are the ones where there is only one answer.

## Three conventions this package commits to

Each of them disagrees with something a reader may be used to, so each
is stated rather than left to be discovered.

- **Eigenvalues and singular values are sorted DESCENDING.** LAPACK's
  `dsyev` and numpy's `eigh` sort eigenvalues *ascending*; everybody's
  singular values are descending. Sorting both descending here makes
  `values(e)[0]` the largest in both modules, makes `truncate(s, k)`
  mean what it looks like, and matches what a caller does next.
- **`lasvd` answers `V`, not `Vᵀ`.** LAPACK and numpy return the
  transpose. Here `V`'s **columns** are the right singular vectors,
  which is how every other matrix in this package is indexed.
- **Symmetry is not checked.** `lachol` and `laeig` read only the lower
  triangle, which is LAPACK's `uplo = 'L'`. Checking symmetry costs
  `n²` comparisons against a tolerance nobody can choose for a caller,
  and a caller who built a covariance matrix knows it is symmetric.
  `lanorm.symmetry_defect` is where the caller who is not sure asks.

## The normal equations are not here, on purpose

`laqr.lstsq` solves `min ||Ax − b||` through `Qᵀb` and back-substitution.
It does **not** form `AᵀA`, and there is no `normal_equations` function.

Solving `AᵀA x = Aᵀb` is the formula every textbook derives and it
squares the condition number. A matrix conditioned at `10⁸` — which a
cubic fit over an unscaled x axis reaches easily — loses every digit it
has through `AᵀA` and keeps half of them through QR. A package that
offered the normal equations beside the QR path would be offering a
footgun with a shorter name.

The same reasoning is why `lalu.inverse` and `lasolve.inverse` each
carry a warning: **to solve a system, use `solve`.** Forming an inverse
and multiplying is three times the work and loses a digit or two. The
two good reasons for an explicit inverse are that a formula needs its
entries — a precision matrix, a Hat matrix — or that it is being
displayed.

## Which least-squares path

Two, and the difference is what happens to a rank-deficient matrix:

| | `lasolve.lstsq` (QR) | `lasolve.lstsq_minimum_norm` (SVD) |
| --- | --- | --- |
| cost | `2mn² − 2n³/3` | several times that |
| full column rank | the unique solution | the same solution |
| rank deficient | **refuses** — infinitely many solutions, and it picks none | the minimum-norm one |
| needs a tolerance | no | yes |

Reach for the QR one. It is faster and it tells you when it is the wrong
choice, which is the property that matters.

## Deliberately not here

- **The general (non-symmetric) eigenproblem.** Its eigenvalues are
  complex even for a real matrix, so its API needs a complex type,
  complex eigenvectors, a left/right convention and a Schur form —
  and novo-lang has no complex type, so all of it would be pairs of
  arrays with a convention in a comment. **A missing row**, named in
  this lane's report.
- **The LQ factorisation** (`dgelqf`), for a matrix with more columns
  than rows. `laqr.factor` refuses that shape with
  `LaUnderdetermined` rather than quietly transposing.
  **A missing row.**
- **Sparse matrices.** A different storage, a different set of
  algorithms and a different package. `NdFloat` is dense by
  construction.
- **Iterative solvers** — conjugate gradient, GMRES, Lanczos. They
  take a matrix-vector *product* rather than a matrix, they need
  preconditioners, and they belong with the sparse row above.
  **A missing row.**
- **Fixed-size 3×3 and 4×4 arithmetic.** geometry-nv has the 2-D
  transforms and the plan's row says not to duplicate them.
- **Any BLAS level-3 surface of its own.** `matmul` is ndarray-nv's;
  this package uses it and does not republish it.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: <pkg>.<module>.<fn>` — which
is the expected result until the bodies land, and is what makes the
suite a description of the interface rather than of nothing.
`novo test --isolate` is the readable form: one verdict per test, naming
the function it stopped at.

| module | public types | functions | constants | implemented |
| --- | --- | --- | --- | --- |
| `lachol` | 1 | 12 | 0 | no |
| `laeig` | 2 | 12 | 1 | no |
| `lafault` | 1 | 3 | 0 | no |
| `lalu` | 2 | 12 | 0 | no |
| `lanorm` | 1 | 10 | 0 | no |
| `laqr` | 1 | 12 | 0 | no |
| `lasolve` | 0 | 10 | 0 | no |
| `lasvd` | 1 | 18 | 1 | no |
| **total** | **9** | **89** | **2** | **no** |
