# linalg-nv

Dense linear algebra is the arithmetic of matrices whose entries are all
stored: solving a system of equations, fitting a line to data, and
taking a matrix apart into simpler factors. This package brings five of
those factorisations to novo-lang, over matrices from
[ndarray-nv](https://novo-lang.org/packages/ndarray-nv). Its numerical
reference is [OpenBLAS](https://www.openblas.net/), which implements the
[LAPACK](https://www.netlib.org/lapack/lug/) and
[BLAS](https://www.netlib.org/blas/) interfaces, and every function here
names the LAPACK routine it mirrors.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a factorisation is, and which five are here

A **factorisation**, or decomposition, writes a matrix as a product of
matrices with simpler shapes. The work of solving a system is done once,
in the factorisation, and every later question is answered from the
factors.

A **triangular** matrix has zeroes on one side of its diagonal. A system
with a triangular matrix is solved one unknown at a time, which is what
makes the factors useful. An **orthogonal** matrix is one whose columns
are perpendicular and of length one; multiplying by it changes no
lengths, so it introduces no error. A matrix is **positive definite**
when it is symmetric and every quantity `xᵀAx` is above zero, which is
true of a covariance matrix. A matrix is **singular** when it has no
inverse.

| Factorisation | Writes the matrix as | Applies to | Answers |
| --- | --- | --- | --- |
| LU | a permutation, a lower and an upper triangle | any square matrix | solve, determinant, inverse |
| QR | an orthogonal matrix and an upper triangle | any matrix with at least as many rows as columns | least squares, residuals, an orthogonal basis |
| Cholesky | a lower triangle and its own transpose | a positive definite matrix | solve, determinant, inverse, and the test for positive definiteness |
| Symmetric eigendecomposition | eigenvalues and orthogonal eigenvectors | a symmetric matrix | the spectrum, the inertia, a matrix power, principal components |
| Singular value decomposition | `U`, the singular values, and `V` | any matrix | rank, pseudoinverse, minimum-norm least squares, condition number, the best low-rank approximation |

An **eigenvalue** of a matrix is a number by which the matrix scales one
particular direction, the **eigenvector**. A **singular value** is the
factor by which a matrix stretches one perpendicular direction into
another. The **rank** is how many independent directions a matrix
actually spans. The **condition number** is the largest singular value
divided by the smallest, and it says how many digits a solve can lose.

The **inertia** of a symmetric matrix is the count of its positive,
negative and zero eigenvalues. The **pseudoinverse** is the closest
thing to an inverse that a non-square or rank-deficient matrix has.

## Install

```
novo pkg add linalg-nv
```

## Example

```novo
use ndfloat
use lalu
use lanorm

fn main() [io]
    // A 2-by-2 matrix, written row by row.
    match ndfloat.of_list([4.0, 3.0, 6.0, 3.0], [2, 2])
        Err(e) => println(e.message())
        Ok(a) =>
            // The cubic work happens here, once.
            match lalu.factor(a)
                Err(e) => println(e.message())
                Ok(lu) =>
                    // The determinant is a product of the factors already held.
                    println("det = ${lalu.determinant(lu)}")

                    // The right-hand side of the system.
                    match ndfloat.of_list([10.0, 12.0], [2])
                        Err(e) => println(e.message())
                        Ok(b) =>
                            // Each solve is quadratic and reuses those factors.
                            match lalu.solve(lu, b)
                                Err(e) => println(e.message())
                                Ok(x)  =>
                                    match lanorm.vector_norm(x)
                                        Err(e) => println(e.message())
                                        Ok(n)  => println("the solution has length ${n}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: linalg-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `lalu` | The LU factorisation with partial pivoting, and the solve, determinant, logarithmic determinant, inverse and reconstruction it answers. |
| `laqr` | The QR factorisation by Householder reflections, the two forms of `Q`, multiplication by `Q` without forming it, the least-squares solve and the residual norm. |
| `lachol` | The Cholesky factorisation, the test for positive definiteness, and the solve, inverse and determinants over it. |
| `laeig` | The symmetric eigendecomposition by cyclic Jacobi, and the spectral radius, condition number, inertia and matrix power it answers. |
| `lasvd` | The singular value decomposition by one-sided Jacobi, and the rank, pseudoinverse, condition number, two-norm, least-squares solve and rank-`k` truncation over it. |
| `lanorm` | Five matrix norms, four vector norms, the dot product, the trace, two condition numbers, and the two measurements that check a matrix's symmetry and a matrix's orthogonality. |
| `lasolve` | One-call forms for a caller with a single system: solve, solve for a positive definite matrix, two least-squares solves, inverse, determinant, rank, pseudoinverse and the two triangular solves. |
| `lafault` | Every reason a factorisation or a solve refuses, as one enum with eleven variants, and three questions to ask of one. |

## How to choose an entry point

**Every factorisation module has the same three parts:** a value that
holds the factors, a `factor` that builds it, and functions that read
it.

**Factor once when you have more than one right-hand side.** `factor` is
the cubic work and `solve` is the quadratic work. A Newton iteration, a
Kalman filter or a simulation step solves against one matrix many times,
and hoisting `factor` out of the loop is the difference between cubic
work once and cubic work every step.

**Use `lasolve` when you have one system and are not coming back.** Each
of its functions factors, answers, and throws the factors away. Each
says in its own documentation which module a repeat caller should use
instead.

**Choose the factorisation by what you know about the matrix.**

| You have | Use |
| --- | --- |
| a square matrix and a right-hand side | `lalu` |
| a symmetric positive definite matrix | `lachol`, which is about twice as fast as LU |
| more equations than unknowns | `laqr` |
| a symmetric matrix and you want its spectrum | `laeig` |
| a matrix that may be rank deficient | `lasvd` |

**There are two least-squares paths, and they differ on a rank-deficient
matrix.**

| | `lasolve.lstsq`, over QR | `lasolve.lstsq_minimum_norm`, over the SVD |
| --- | --- | --- |
| Full column rank | the unique solution | the same solution |
| Rank deficient | refused | the solution of smallest length |
| Needs a tolerance | no | yes |

Reach for the QR one first. It is faster, and it says when it is the
wrong choice.

## The rules a user needs

1. **Eigenvalues and singular values come back in descending order.**
   LAPACK's `dsyev` and NumPy's `eigh` sort eigenvalues ascending.
   Sorting both descending here makes the first element the largest in
   both modules, and makes `lasvd.truncate` keep the largest `k`.
2. **`lasvd.v` answers `V`, not its transpose.** LAPACK and NumPy return
   the transpose. Here `V`'s columns are the right singular vectors,
   which is how every other matrix in this package is indexed.
3. **Symmetry is not checked.** `lachol` and `laeig` read the lower
   triangle only, which is LAPACK's `uplo = 'L'`. A check would cost a
   comparison per entry against a tolerance no library can choose for a
   caller. `lanorm.symmetry_defect` is where a caller who is unsure
   asks.
4. **To solve a system, use `solve` and not `inverse`.** Forming an
   inverse and multiplying by it is about three times the work and loses
   a digit or two. An explicit inverse is right when a formula needs its
   entries, as a precision matrix or a hat matrix does, or when it is
   being displayed.
5. **The normal equations are not here.** `laqr.lstsq` solves the
   least-squares problem through `Q` transpose times `b` and back
   substitution. Forming `AᵀA` squares the condition number, so a matrix
   conditioned at ten to the eighth loses every digit it has that way
   and keeps half of them through QR.
6. **`laqr.factor` refuses a matrix with more columns than rows.** The
   refusal is `LaUnderdetermined`. It does not transpose the matrix for
   you.
7. **A rank and a pseudoinverse need a tolerance**, which is the
   singular value below which a direction counts as absent.
   `lasvd.default_rank_tolerance` is the usual choice, computed from the
   matrix's size and its largest singular value.
8. **`lalu.log_abs_determinant` exists because a determinant overflows.**
   The determinant of a large matrix runs past the range of a float. The
   logarithmic form answers a sign and a logarithm separately.
9. **The two Jacobi factorisations iterate, and can refuse to
   converge.** `laeig` and `lasvd` sweep at most `LAEIG_SWEEPS` and
   `LASVD_SWEEPS` times, which is 30 each. `factor_with_sweeps` sets
   another limit. Failure is `LaNoConvergence`, carrying the sweep count
   and the residual.
10. **A refusal names the position.** `LaSingular` carries the step and
    the pivot that was too small. `LaNotPositiveDefinite` carries the
    step and the value that was not above zero.
11. **`matmul` is not republished here.** Multiplying two matrices is
    ndarray-nv's `ndfloat.matmul`, and this package uses it.

## Which LAPACK routine each function mirrors

A result that disagrees with LAPACK is this package's fault, so each
function names the routine to check it against.

| This package | LAPACK | What it is |
| --- | --- | --- |
| `lalu.factor` | `dgetrf` | LU with partial pivoting |
| `lalu.solve` | `dgetrs` | solve from the factors |
| `lalu.inverse` | `dgetri` | inverse from the factors |
| `lanorm.condition_estimate` | `dgecon` | one-norm condition estimate |
| `laqr.factor` | `dgeqrf` | Householder QR |
| `laqr.q` | `dorgqr` | form `Q` explicitly |
| `laqr.apply_q` | `dormqr` | multiply by `Q` without forming it |
| `laqr.lstsq` | `dgels` | full-rank least squares |
| `lachol.factor` | `dpotrf` | Cholesky |
| `lachol.solve` | `dpotrs` | solve from the factors |
| `lachol.inverse` | `dpotri` | inverse from the factors |
| `laeig.factor` | `dsyevj` | symmetric eigendecomposition by Jacobi |
| `lasvd.factor` | `dgesvj` | singular value decomposition by one-sided Jacobi |
| `lasvd.lstsq` | `dgelss` | minimum-norm least squares |
| `lanorm.norm` | `dlange` | the matrix norms other than the two-norm |
| `lanorm.vector_norm` | `dnrm2` | scaled Euclidean norm |
| `lanorm.dot` | `ddot` | dot product |
| `lasolve.solve_lower`, `.solve_upper` | `dtrsv` | triangular solve |

Two routines are deliberately not the fastest LAPACK offers. The
symmetric eigendecomposition uses cyclic Jacobi rather than the
tridiagonal QR iteration of `dsyev`, and the singular value
decomposition uses one-sided Jacobi rather than the bidiagonal method of
`dgesdd`. Jacobi computes small eigenvalues and singular values to high
relative accuracy on a matrix whose scale varies across its columns,
which is any covariance matrix and any design matrix in mixed units. The
result types say nothing about which algorithm produced them, so a
faster path can be substituted later without changing a caller.

## What is not included

- **The eigendecomposition of a non-symmetric matrix.** Its eigenvalues
  are complex even when the matrix is real, so it needs a complex type,
  which novo-lang does not have. Every part of it would be pairs of
  arrays with a convention in a comment.
- **The LQ factorisation**, for a matrix with more columns than rows.
  See rule 6.
- **Sparse matrices.** They are a different storage and a different set
  of algorithms. An `NdFloat` is dense by construction.
- **Iterative solvers**, such as conjugate gradient, GMRES and Lanczos.
  They take a function that multiplies by the matrix rather than the
  matrix itself, and they need preconditioners.
- **Fixed-size three-by-three and four-by-four arithmetic.**
  [geometry-nv](https://novo-lang.org/packages/geometry-nv) has the
  two-dimensional transforms.
- **A BLAS surface of its own.** See rule 11.
- **A microcontroller build.** Every matrix here is an ndarray-nv array,
  which is a heap value, and a build for a device with no heap allocator
  refuses a list literal (SPEC section 14.4). Nothing here is claimed to
  build for such a device, and there is no `tests/embedded_probe.nv`.

## Related packages

- [ndarray-nv](https://novo-lang.org/packages/ndarray-nv) is the matrix.
  Every function here takes or answers one of its arrays, and its
  `matmul` is the matrix product.
- [stats-nv](https://novo-lang.org/packages/stats-nv) produces the
  covariance matrices this package factors, and is where a summary of a
  sample lives.
- [geometry-nv](https://novo-lang.org/packages/geometry-nv) is
  fixed-size two-dimensional transforms, for a program that wants a
  rotation rather than a decomposition.
- `std.array` in the standard library wraps a foreign numeric array.
  Every one of its methods declares an effect, and nothing in this
  package does.

## Tests

```bash
novo test tests/linalg_tests.nv       # 26 tests: the identities and the hand factorisations
```

A factorisation of an arbitrary matrix is not unique. The signs of `Q`'s
columns, the direction of an eigenvector and the sign of a singular
vector are all free, so a suite that recorded LAPACK's output for a
random matrix would be asserting one correct answer against another. The
suite therefore asserts three kinds of thing.

The first is matrices factored by hand, where there is only one answer:
the identity, a diagonal matrix, a two-by-two whose LU and inverse fit
on one line, and the swap matrix that needs a pivot on the very first
step.

The second is identities every decomposition owes whatever it produced.
The factors multiply back to the original matrix, `Q` is orthogonal, the
eigenvalues sum to the trace and multiply to the determinant, the
singular values are non-negative and descending, the rank-`k`
truncation of a rank-`k` matrix is exact, and the pseudoinverse of a
full-rank square matrix is its inverse.

The third is the two conventions that disagree with NumPy, in rules 1
and 2.

Two tolerances are used. A direct arithmetic identity, such as a norm or
a trace, is asserted to within 1e-12, because it is a handful of
operations above a floor near 1e-16. Anything that has been through a
factorisation is asserted to within 1e-10. A backward-stable
decomposition of a matrix of these sizes has an error near 1e-14, and a
wrong implementation is wrong by a whole unit rather than by 1e-9.

The tests compile today and fail at run, each on the
`not implemented: linalg-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `laeig.LAEIG_SWEEPS`, `lasvd.LASVD_SWEEPS` | yes (they are constants) |
| `lalu.LaLu`, `.LaLogDet`, `laqr.LaQr`, `lachol.LaChol` | declared |
| `laeig.LaEigen`, `.LaInertia`, `lasvd.LaSvd`, `lanorm.LaNorm`, `lafault.LaFault` | declared |
| `lalu.factor`, `.order`, `.lower`, `.upper`, `.permutation`, `.permutation_indices`, `.reconstruct` | no |
| `lalu.solve`, `.solve_matrix`, `.determinant`, `.log_abs_determinant`, `.inverse` | no |
| `laqr.factor`, `.rows`, `.cols`, `.q`, `.q_thin`, `.r`, `.reconstruct` | no |
| `laqr.apply_q`, `.apply_q_transpose`, `.lstsq`, `.residual_norm`, `.solve_r` | no |
| `lachol.factor`, `.is_positive_definite`, `.order`, `.lower`, `.upper`, `.reconstruct` | no |
| `lachol.solve`, `.solve_matrix`, `.inverse`, `.determinant`, `.log_determinant`, `.apply_lower` | no |
| `laeig.factor`, `.factor_with_sweeps`, `.values_only`, `.order`, `.values`, `.vectors`, `.vector`, `.reconstruct` | no |
| `laeig.spectral_radius`, `.condition`, `.inertia`, `.power` | no |
| `lasvd.factor`, `.factor_with_sweeps`, `.values_only`, `.count`, `.rows`, `.cols`, `.reconstruct` | no |
| `lasvd.singular_values`, `.u`, `.v`, `.u_full` | no |
| `lasvd.rank`, `.default_rank_tolerance`, `.condition`, `.pseudoinverse`, `.lstsq`, `.truncate`, `.norm_2` | no |
| `lanorm.norm`, `.vector_norm`, `.vector_norm_p`, `.vector_norm_inf`, `.dot`, `.trace` | no |
| `lanorm.condition`, `.condition_estimate`, `.symmetry_defect`, `.orthogonality_defect` | no |
| `lasolve`'s ten one-call forms | no |
| `lafault`'s eleven variants, its three predicates and its `Error` implementation | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
