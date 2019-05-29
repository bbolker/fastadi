---
title: "computational details of low-rank matrix completion via adaptive-impute"
author: "Alex Hayes"
date: "2019-05-29"
output: pdf_document
bibliography: references.bib
link-citations: true
urlcolor: blue
header-includes:
  \usepackage[linesnumbered,ruled,vlined]{algorithm2e}
  \DeclareMathOperator{\diag}{diag}
  \DeclareMathOperator{\trace}{trace}
  \DeclareMathOperator{\sign}{sign}
  \linespread{1.25}
  \usepackage{helvet}
  \renewcommand{\familydefault}{\sfdefault}
---



\newcommand \bv {\mathbf{v}}
\newcommand \bu {\mathbf{u}}
\newcommand \bl {\boldsymbol{\lambda}}

## Abstract

TODO: motivate ugh this is a show

This paper is a tutorial covering the computational details associated with computing the low-rank rank adaptive imputations for matrices described in [@cho_asymptotic_2015; @cho_intelligent_2018]. This work extends previous data adaptive matrix imputation strategies such as that of [@mazumder_spectral_2010], and has better performance while eliminating tuning parameters.

The tutorial proceeds in three parts. First, we introduce the imputation algorithms, useful tidbits of linear algebra and the R package `Matrix`, which we will use to illustrate computations. After a naive initial implementation which eats up lots of memory, we demonstrate a memory-efficient implementation. Finally, we extend this memory-efficient implementation to the partially observed matrices that possibly have a large number of observed zeros.

[@bates_introduction_2005; @cho_asymptotic_2015; @cho_intelligent_2018; @maechler_2nd_2006; @mazumder_spectral_2010; @bro_resolving_2007]


## Notation & Algorithm

There are two steps to computing the low rank approximation of [@cho_intelligent_2018]. First we use compute an initial low rank estimate with the `AdaptiveInitialize` algorithm. This step is essentially a debiased SVD. We then use the initial solution as a seed for the `AdaptiveImpute` algorithm, a form of iterative SVD thresholding with a data adaptive thresholding parameter.

The input to these algorithms is a partially observed matrix $M$, and $r$, the desired rank of the low-rank approximation. We define $Y$ to be an indicator matrix that tells us whether or not we observed a particular value of $M$.

\begin{algorithm}
\linespread{1.6}\selectfont
\DontPrintSemicolon
\KwIn{$M, Y$ and $r$}
$\hat p \gets \frac{1}{nd} \sum_{i=1}^n \sum_{j=1}^d Y_{ij}$ \;
$\Sigma_{\hat p} \gets M^T M - (1 - \hat p) \diag(M^T M)$ \;
$\Sigma_{t \hat p} \gets M M^T - (1 - \hat p) \diag(M M^T)$ \;
$\hat V_i \gets \bv_i(\Sigma_{\hat p})$ for $i = 1, ..., r$ \;
$\hat U_i \gets \bu_i(\Sigma_{t \hat p})$ for $i = 1, ..., r$ \;
$\tilde \alpha \gets \frac{1}{d - r} \sum_{i=r+1}^d \bl_i (\Sigma_{\hat p})$ \;
$\hat \lambda_i \gets \frac{1}{\hat p} \sqrt{\bl_i (\Sigma_{\hat p}) - \tilde \alpha}$ for $i = 1, ..., r$ \;
$\hat s_i \gets \sign(\langle \hat V_i, \bv_i (M) \rangle) \sign(\langle \hat U_i, \bu_i (M) \rangle)$ for $i = 1, ..., r$ \;
$\hat \lambda_i \gets \hat s_i \cdot \hat \lambda_i$ \;
\Return{$\hat \lambda_i, \hat U_i, \hat V_i$ for $i = 1, ..., r$}\;
\caption{\texttt{AdapativeInitialize}}
\end{algorithm}

Here $\bu_i, \bl_i, \bv_i$ are functions that return the $i^{th}$ left singular vector, singular value, and right singular value, respectively. $\tilde \alpha$ is the data adaptive thresholding parameter. Note that it is the average of the uncalculated singular values, and is thus positive.

In line 8, $\langle , \rangle$ denotes the Frobenius inner norm. That is, for vectors $x, y$ and matrices $A, B$:

\begin{align}
\langle x, y \rangle &= x^T y = \sum_{i} x_i y_i \\
\langle A, B \rangle &= \sum_{i, j} A_{ij} B_{ij} = \sum_{ij} A \odot B \\
\end{align}

We use $A \odot B$ to mean the elementwise (Hadamard) product of matrices $A$ and $B$.

We can use the output of `AdaptiveInitialize` to construct a low rank approximation to $M$ via:

\begin{align}
\hat M = \sum_{i=1}^r \hat \lambda_i \hat U_i \hat V_i^T
\end{align}

Next up is `AdaptiveImpute`:

\begin{algorithm}
\linespread{1.6}\selectfont
\DontPrintSemicolon
\KwIn{$M, Y, r$ and $\varepsilon > 0$}

$Z^{(1)} \gets \texttt{AdaptiveInitialize}(M, Y, r)$ \;

\Repeat{$\| Z_{t+1} - Z_t \|^2_F / \| Z_{t+1} \|_F$}{
  
  $\tilde M^{(t)} \gets P_\Omega (M) + P_\Omega^\perp (Z_t)$ \;
  $\hat V_i^{(t)} \gets \bv_i(\tilde M^{(t)})$ for $i = 1, ..., r$ \;
  $\hat U_i^{(t)} \gets \bu_i(\tilde M^{(t)})$ for $i = 1, ..., r$ \;
  $\tilde \alpha^{(t)} \gets \frac{1}{d - r} \sum_{i=r+1}^d \bl_i^2 (\tilde M^{(t)})$ \;
  $\hat \lambda_i^{(t)} \gets \sqrt{\bl_i^2 (\tilde M^{(t)}) - \tilde \alpha^{(t)}}$ for $i = 1, ..., r$ \;
  $Z^{(t+1)} \gets \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}$ \;
  $t \gets t + 1$ \;
}
\Return{$\hat \lambda_i^{(t)}, \hat U_i^{(t)}, \hat V_i^{(t)}$ for $i = 1, ..., r$}\;
\caption{\texttt{AdaptiveImpute}}
\end{algorithm}

Here we again have some new notation. First we define

\begin{align}
P_\Omega(A) &= A \odot Y \\
P_\Omega^\perp (A) &= A \odot (1 - Y)
\end{align}

These are the projecttion of a matrix $A$ onto the observed of $M$ and the unobserved elements of $M$, respectively. Similar $\Omega$ is the set of all pairs $(i, j)$ such that $M_{i, j}$ is observed. $\bl_i^2 (A)$ is a function that returns the $i^{th}$ *squared* singular value of $A$ (i.e. $\bl_i^2 (A) = \left(\bl_i (A)\right)^2$). Finally $\Vert \cdot \Vert_F$ is the Frobenius norm of a matrix.

## Pre-requisites

### The Matrix package

If you haven't used the `Matrix` package before, we recommend reading [this introduction][matrix_intro] as well as this [2nd introduction][matrix_2nd_intro].

[matrix_intro]: https://cran.r-project.org/web/packages/Matrix/vignettes/Intro2Matrix.pdf
[matrix_2nd_intro]: https://cran.r-project.org/web/packages/Matrix/vignettes/Intro2Matrix.pdf


```r
library(Matrix)

set.seed(27)

# create a random 8 x 12 sparse matrix with 30 nonzero entries
M <- rsparsematrix(8, 12, nnz = 30)
M
```



```r
summary(M)
```

TODO: different storage formats for sparse matrices. CSC, triplet, symmetric. for the most part `Matrix` does the right thing for you. The triplet form will be most important for us later on when we right out some matrix multiplications by hand.


```r
# note that Matrix objects are S4 classes so we access their
# slots using the @ symbol
class(M)

M@x  # vector of values in M
M@i  # corresponding row indices
```

coercing


```r
# we want a tripletMatrix, but *don't* specify the subclass
M2 <- as(M, "TsparseMatrix") 

# i.e. the following is bad style and will possibly break
M2 <- as(M, "dgTMatrix")

# but in triplet format we can get the corresponding column indices
M2@j  
```

We will repeatedly calculate squared Frobenious norms throughout this tutorial. It's important to know that there are *many, many* ways to calculate this norm. We will almost always calculate the squared Frobenius norm of a sparse Matrix `M` via `sum(M@x^2)`. That said, these are all equivalent:


```r
M^2    # square each element in M elementwise, return as sparse matrix
M@x^2  # square each element in M elementwise, return as vector of nonzeros

# the second version is much faster
bench::mark(
  sum(M@x^2),
  sum(M^2),
  sum(colSums(M^2)),
  norm(M, type = "F")^2,
  sum(M * M),
  iterations = 20
)
```

The projections $P_\Omega (A)$ and $P_\Omega^\perp (A)$ where $\Omega$ indicates the observed elements of a matrix $M$ and $A$ is another matrix with the same dimensions as $M$.


```r
y <- as(M, "lgCMatrix")  # indicator matrix only
all.equal(y * M, M)  # don't lose anything multiplying by indicators

A <- matrix(1:(8*12), 8, 12)

all.equal(dim(A), dim(M))  # appropriate to practice projections with

# Omega indicates whether an entry of M was observed

# P_Omega (A)
A * y

!y

# P_Omega^perp (A): NOTE: this results in a *dense* matrix
A * (1 - y)

all(A * y + A * (1 - y) == A)  # can recover A from both projections together
```

matrix-matrix crossproducts


```r
bench::mark(
  crossprod(M),     # dsCMatrix -- most specialized class, want this
  crossprod(M, M),  # dgCMatrix
  t(M) %*% M,       # dgCMatrix
  check = FALSE
)
```

matrix-vector crossproducts


```r
# TODO
```


the `drop()` function helps us manage dimensions


```r
one_col <- matrix(1:4)
one_row <- matrix(5:8, nrow = 1)

drop(one_col)
drop(one_row)

c(one_col)  # same thing, less explicit. use drop to be explicit
```


```r
# TODO: drop vs drop0 vs drop1
```


diagonal of crossproduct


```r
v_sign == colSums(svd_M$v * v_hat)
diag(t(svd_M$v) %*% v_hat)
diag(crossprod(svd_M$v, v_hat))

bench::mark(
  colSums(svd_M$v * v_hat),
  crossprod(rep(1, d), svd_M$v * v_hat),
  iterations = 50,
  check = FALSE
)

# write a diag_crossprod helper
```


```r
rhos <- matrix(1:12, ncol = 4, byrow = TRUE)

bench::mark(
  diag(crossprod(rhos)),
  diag(t(rhos) %*% rhos),
  colSums(rhos * rhos),
  crossprod(rep(1, nrow(rhos)), rhos^2),
  check = FALSE                            
)


rhos <- matrix(1:12, ncol = 4, byrow = TRUE)

bench::mark(
  diag(tcrossprod(rhos)),
  diag(crossprod(t(rhos))),
  diag(rhos %*% t(rhos)),
  rowSums(rhos * rhos),
  check = FALSE
)
```

what we get from `eigen()` and `svd()`: slightly different stuff: `u`, `d` and `v` versus `values` and `vectors`

NOTE: RSpectra *only* does truncated decompositions. if you want the full decomposition, you have to use base R stuff. different algos.

## Brief aside in sign ambiguity

yada yada yada the signs of the left and right singular vectors are not identified in SVD

A more elegant solution as proposed in @bro_resolving_2007 and used in Karl's paper is to take inner products

NOTE TO SELF: identifying the signs of a single SVD is a much harder task than comparing two SVDs and seeing if they are the same up to sign differences. we only need to check if they are the same up to sign differences.


```r
set.seed(17)
M <- rsparsematrix(8, 12, nnz = 30) # small example, not very sparse

# number of singular vectors to compute
k <- 4

s <- svd(M, k, k)
s2 <- svds(M, k, k)

# irritating: svd() always gives you all the singular values even if you 
# only request the first K singular vectors
s$u %*% diag(s$d[1:k]) %*% t(s$v)

# based on the flip_signs function of
# https://stats.stackexchange.com/questions/134282/relationship-between-svd-and-pca-how-to-use-svd-to-perform-pca
equal_svds <- function(s, s2) {
  
  # svd() always gives you all the singular values, but we only
  # want to compare the first k
  k <- ncol(s$u)
  
  # the term sign(s$u) * sign(s2$u) performs a sign correction
  
  # isTRUE because output of all.equal is not a boolean, it's something
  # weird when the inputs aren't equal. lol why
  
  u_ok <- isTRUE(
    all.equal(s$u, s2$u * sign(s$u) * sign(s2$u), check.attributes = FALSE)
  )
  
  v_ok <- isTRUE(
    all.equal(s$v, s2$v * sign(s$v) * sign(s2$v), check.attributes = FALSE)
  )
  
  d_ok <- isTRUE(all.equal(s$d[1:k], s2$d[1:k], check.attributes = FALSE))
  
  u_ok && v_ok && d_ok
}
```

### Linear algebra facts

Throughout these computations, we will repeatedly use several key facts about eigendecompositions, singular value decompositions (SVD) and the relationship between the two.

https://en.wikipedia.org/wiki/Gramian_matrix X'X -- positive semi-def, so the singular values of M'M are the same as the eigenvalues

question: if A, B positive, is
sum(svd(A - B)$d) == sum(svd(A)$d) - sum(svd(B)$d)

answer: NO! can't split into two easy computations and then combine them
possibly use this to get some sort of bound?

think about what happens as p_hat -> 0

A key observation here is that $M^T M$ and

Fact: sum of squared singular values is `trace(A^T A)`
https://math.stackexchange.com/questions/2281721/sum-of-singular-values-of-a-matrix

Fact: for symmetric positive definite matrices the eigendecomp is equal to the singular value decomp

Fact: sum of eigenvalues of M is equal to trace(M)

Consequence: for pos def symmetric M the sum of the singular values is trace(M) as well

Again computing `alpha` deserves some explanation.

- reference: https://math.stackexchange.com/questions/1463269/how-to-obtain-sum-of-square-of-eigenvalues-without-finding-eigenvalues
- Frobenius norm (A) = trace(crossprod(A))
- TODO: how we know this thing is strictly positive to prevent sqrt() from exploding


```r
## STOPPED HERE: WHY ARE the following not the same?
  isSymmetric(sigma_p)
  eigen(sigma_p)$values
  sum(diag(sigma_p))
  sum(svd(sigma_p)$d)
  sum(eigen(sigma_p)$values)
  
  # let's think just about the first term for a moment
  sum(diag(MtM / p_hat^2))
  sum(svd(MtM / p_hat^2)$d)
  
  sum(colSums(M^2 / p_hat^2))
  
  # has some negative eigenvalues
  
  # Fact: for a symmetric matrix, the singular values are the *absolute* values
  # of the eigenvalues
  
  # https://www.mathworks.com/content/dam/mathworks/mathworks-dot-com/moler/eigs.pdf
  
  eigen(sigma_p)$values
  sum(eigen(sigma_p)$values)
  sum(abs(eigen(sigma_p)$values))
  sum(svd(sigma_p)$d)
  sum(abs(diag(sigma_p)))
  
  # those agree so what about the second term
  
  # note to self: alpha should be positive
  # issue karl ran into:
  # https://math.stackexchange.com/questions/381808/sum-of-eigenvalues-and-singular-values
  # how to get the sum of singular values itself (start):
  # https://math.stackexchange.com/questions/569989/sum-of-singular-values-of-ab
  
  # options when sigma_p is not positive definite:
  # - calculate the full SVD
  # - set alpha to zero (don't truncate the singular values)
  # - this lower bounds the average of the remaining singular values
  # -
  
  # positive semi-definite is enough since symmetric and eigen/singular values
  # of zero don't matter
  
  # this is only an issue in the initialization. in the iterative updates
  # we use the squared singular values, which we can more easily calculate
  # the sum of
  
  # ask Karl what he wants to do about this: computing a full SVD is gonna be really expensive.
```

FACT: sum(diag(crossprod(M))) == sum(M^2)

## Reference implementation


```r
library(RSpectra)
library(Matrix)
```

<!-- ```{#AdaptiveImpute .R .numberLines} -->


```r
adaptive_initialize <- function(M, r) {
  
  # TODO: ignores observed zeros!
  p_hat <- nnzero(M) / prod(dim(M))  # line 1
  
  MtM <- crossprod(M)
  MMt <- tcrossprod(M)
  
  # need to divide by p^2 from Cho et al 2016 to get the "right"
  # singular values / singular values on a comparable scale
  
  # both of these matrices are symmetric, but not necessarily positive
  # this has important implications for the SVD / eigendecomp relationship
  
  sigma_p <- MtM / p_hat^2 - (1 - p_hat) * diag(diag(MtM))  # line 2
  sigma_t <- MMt / p_hat^2 - (1 - p_hat) * diag(diag(MMt))  # line 3
  
  # crossprod() and tcrossprod() return dsCMatrix objects,
  # sparse matrix objects that know they are symmetric
  
  # unfortunately, RSpectra doesn't support dsCMatrix objects,
  # but does support dgCMatrix objects, a class representing sparse
  # but not symmetric matrices
  
  # support for dsCMatrix objects in RSpectra is on the way,
  # which will eliminate the need for the following coercions.
  # see: https://github.com/yixuan/RSpectra/issues/15
  
  sigma_p <- as(sigma_p, "dgCMatrix")
  sigma_t <- as(sigma_t, "dgCMatrix")
  
  svd_p <- svds(sigma_p, r)  # TODO: is eigs_sym() faster?
  svd_t <- svds(sigma_t, r)
  
  v_hat <- svd_p$v  # line 4
  u_hat <- svd_t$u  # line 5
  
  n <- nrow(M)
  d <- ncol(M)
  
  # NOTE: alpha is incorrect due to singular values and eigenvalues
  # being different when sigma_p is not positive
  
  alpha <- (sum(diag(sigma_p)) - sum(svd_p$d)) / (d - r)  # line 6
  lambda_hat <- sqrt(svd_p$d - alpha) / p_hat             # line 7
  
  svd_M <- svds(M, r)
  
  v_sign <- crossprod(rep(1, d), svd_M$v * v_hat)
  u_sign <- crossprod(rep(1, n), svd_M$u * u_hat)
  s_hat <- drop(sign(v_sign * u_sign))
  
  lambda_hat <- lambda_hat * s_hat  # line 8
  
  list(u = u_hat, d = lambda_hat, v = v_hat)
}
```


It's worth commenting on computation of `alpha` and `s_hat`.

When we compute `alpha` in line 20 `adaptive_initialize()`, we don't want to do the full eigendecomposition of $\Sigma_{\hat p}$ since that could take a long time, so we use trick and recall that the trace of a matrix (the sum of it's diagonal elements) equals the sum of all the eigenvalues. Then we subtract off the first $r$ eigenvalues, which we do compute, and are left with $\sum_{i = r + 1}^d \lambda_i(\Sigma_{\hat p})$.

<!-- ```{#AdaptiveImpute .R .numberLines} -->

```r
adaptive_impute <- function(M, r, epsilon = 1e-7) {
  
  s <- adaptive_initialize(M, r)
  Z <- s$u %*% diag(s$d) %*% t(s$v)  # line 1
  delta <- Inf
  
  while (delta > epsilon) {
    
    y <- as(M, "lgCMatrix")  # indicator if entry of M observed
    M_tilde <- M + Z * (1 - y)  # line 3
    
    svd_M <- svds(M_tilde, r)
    
    u_hat <- svd_M$u  # line 4
    v_hat <- svd_M$v  # line 5
    
    d <- ncol(M)
    
    alpha <- (sum(M_tilde^2) - sum(svd_M$d^2)) / (d - r)  # line 6
    
    lambda_hat <- sqrt(svd_M$d^2 - alpha)  # line 7
    
    Z_new <- u_hat %*% diag(lambda_hat) %*% t(v_hat)
    
    delta <- sum((Z_new - Z)^2) / sum(Z^2)
    Z <- Z_new
    
    print(glue::glue("delta: {round(delta, 8)}, alpha: {round(alpha, 3)}"))
  }
  
  Z
}
```

Finally we can do a minimal sanity check and see if this code even runs, and see if we are recovering something close-ish to implanted low-rank structure.


```r
n <- 500
d <- 100
r <- 5

A <- matrix(runif(n * r, -5, 5), n, r)
B <- matrix(runif(d * r, -5, 5), d, r)
M0 <- A %*% t(B)

err <- matrix(rnorm(n * d), n, d)
Mf <- M0 + err

p <- 0.3
y <- matrix(rbinom(n * d, 1, p), n, d)
dat <- Mf * y

init <- adaptive_initialize(dat, r)
filled <- adaptive_impute(dat, r)
```

## Low-rank implementation

The reference implementation has some problems. As our data matrix $M$ gets larger, we can no longer fit the dense representation of $\hat M$ and $Z^{(t)}$ into memory. Instead, we need to work with just the low rank components $\hat \lambda, \hat U$ and $\hat V$.

This leads us to following implementation:


```r
low_rank_adaptive_initialize <- function(M, r) {
  
  M <- as(M, "dgCMatrix")
  
  p_hat <- nnzero(M) / prod(dim(M))  # line 1
  
  # NOTE: skip explicit computation of line 2
  # NOTE: skip explicit computation of line 3
  
  eig_p <- eigen_helper(M, r)
  eig_t <- eigen_helper(t(M), r)
  
  lr_v_hat <- eig_p$vectors  # line 4
  lr_u_hat <- eig_t$vectors  # line 5
  
  d <- ncol(M)
  n <- nrow(M)
  
  # NOTE: alpha is again incorrect since we work with eigenvalues
  # rather than singular values here
  sum_eigen_values <- sum(M@x^2) / p^2 - (1 - p) * sum(colSums(M^2))
  lr_alpha <- (sum_eigen_values - sum(eig_p$values)) / (d - r)  # line 6
  
  lr_lambda_hat <- sqrt(eig_p$values - lr_alpha) / p_hat  # line 7
  
  # TODO: Karl had another sign computation here that he said was faster
  # but it wasn't documented anywhere, so I'm going with what was in the 
  # paper
  
  lr_svd_M <- svds(M, r)
  
  # v_hat is d by r
  lr_v_sign <- crossprod(rep(1, d), lr_svd_M$v * lr_v_hat)
  lr_u_sign <- crossprod(rep(1, n), lr_svd_M$u * lr_u_hat)
  lr_s_hat <- c(sign(lr_v_sign * lr_u_sign))  # line 8
  
  lr_lambda_hat <- lr_lambda_hat * lr_s_hat
  
  list(u = lr_u_hat, d = lr_lambda_hat, v = lr_v_hat)
}
```

What does `eigen_helper()` do? Describe the return object (also do this for svds)


```r
# Take the eigendecomposition of t(M) %*% M - (1 - p) * diag(t(M) %*% M)
# using sparse computations only
eigen_helper <- function(M, r) {
  eigs_sym(
    Mx, r,
    n = ncol(M),
    args = list(
      M = M,
      p = nnzero(M) / prod(dim(M))
    )
  )
}

# compute (t(M) %*% M / p^2 - (1 - p) * diag(diag(t(M) %*% M))) %*% x
# using sparse operations

# TODO: divide the second term by p^2 like in the reference implementatio
Mx <- function(x, args) {
  drop(
    crossprod(args$M, args$M %*% x) / args$p^2 - (1 - args$p) * Diagonal(ncol(args$M), colSums(args$M^2)) %*% x
  )
}
```

Now we check that `Mx` works


```r
x <- rnorm(12)
p <- 0.3
out <- (t(M) %*% M / p^2 - (1 - p) * diag(diag(t(M) %*% M))) %*% x
out2 <- Mx(x, args = list(M = M, p = p))
all.equal(as.matrix(out), as.matrix(out))
```


TODO: update alg description to divide by $p^2$ to get the right singular values

Quickly check that the components works before we try the code that integrates them all together



Finally, sanity check this by comparing to the reference implementation. These don't agree, which isn't great:


```r
lr_init <- low_rank_adaptive_initialize(dat, r)

# some weird stuff is happening with the singular values but I'm
# going to not worry about it for the time being

equal_svds(init, lr_init)
```

## Space-efficient adaptive impute

TODO: figure out the actual space complexity

Recall the algorithm looks like

\begin{align}
\hat M = \sum_{i=1}^r \hat s_i  \hat \lambda_i \hat U_i \hat V_i^T
\end{align}

\begin{algorithm}
\linespread{1.6}\selectfont
\DontPrintSemicolon
\KwIn{$M, y, r$ and $\varepsilon > 0$}

$Z^{(1)} \gets \texttt{AdaptiveInitialize}(M, y, r)$ \;

\Repeat{$\| Z_{t+1} - Z_t \|^2_F / \| Z_{t+1} \|_F$}{
  
  $\tilde M^{(t)} \gets P_\Omega (M) + P_\Omega^\perp (Z_t)$ \;
  $\hat V_i^{(t)} \gets \bv_i(\tilde M^{(t)})$ for $i = 1, ..., r$ \;
  $\hat U_i^{(t)} \gets \bu_i(\tilde M^{(t)})$ for $i = 1, ..., r$ \;
  $\tilde \alpha^{(t)} \gets \frac{1}{d - r} \sum_{i=r+1}^d \bl_i^2 (\tilde M^{(t)})$ \;
  $\hat \lambda_i^{(t)} \gets \sqrt{\bl_i^2 (\tilde M^{(t)}) - \tilde \alpha^{(t)}}$ for $i = 1, ..., r$ \;
  $Z^{(t+1)} \gets \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}$ \;
  $t \gets t + 1$ \;
}
\Return{$\hat \lambda_i^{(t)}, \hat U_i^{(t)}, \hat V_i^{(t)}$ for $i = 1, ..., r$}\;
\caption{\texttt{AdapativeImpute}}
\end{algorithm}

Now we need two things:

1. The SVD of $\tilde M^{(t)}$
2. (Certain sums of) the squared singular values.

### Sums of squared singular values

For a matrix $A$, the sum of squared singular values (denoted by $\lambda_i$) equals the squared frobenius norm:

\begin{align}
\sum_{i=1}^{\min(n, d)} \lambda_i^2 = ||A||_F^2 = \trace(A^T A)
\end{align}

Also note that 

\begin{align}
||A + B||_F^2 = ||A||_F^2 + ||B||_F^2 + 2 \cdot \langle A, B \rangle_F
\end{align}

Now we consider $\tilde M^{(t)}$. Suppose that unobserved values of $M$ are set to zero, as is the case for $M$ stored in a sparse matrix representation

\begin{align}
\tilde M^{(t)} &= P_\Omega(M) + P_\Omega^\perp (Z_t) \\
&= P_\Omega(M) + P_\Omega^\perp \left(
  \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
  \right)
\end{align}

Now we need

\begin{align}
||\tilde M^{(t)}||_F^2 
&= \left \Vert
  P_\Omega(M) + P_\Omega^\perp \left(
    \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
    \right)
  \right \Vert_F^2 \\
&= \left \Vert P_\Omega(M) \right \Vert_F^2 
  + \left \Vert P_\Omega^\perp \left(
      \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
    \right)
  \right \Vert_F^2
  + 2 \cdot \left \langle P_\Omega(M), P_\Omega^\perp \left(
      \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
    \right) \right \rangle_F \\
&= \left \Vert P_\Omega(M) \right \Vert_F^2 
  + \left \Vert P_\Omega^\perp \left(
      \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
    \right)
  \right \Vert_F^2
\end{align}

Where the cancellation in the final line follows because

\begin{align}
\left \langle P_\Omega(M), P_\Omega^\perp
  \left(
    \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
  \right)
\right \rangle_F
= \sum_{i, j} P_\Omega(M)_{ij} \cdot P_\Omega^\perp (Z_t)_{ij}
= \sum_{i, j} 0
= 0
\end{align}

Now we need one more trick, which is that

\begin{align}
\left \Vert Z_t \right \Vert_F^2
= \left \Vert P_\Omega (Z_t) + P_\Omega^\perp (Z_t) \right \Vert_F^2
= \left \Vert P_\Omega (Z_t) \right \Vert_F^2 + 
  \left \Vert P_\Omega^\perp (Z_t) \right \Vert_F^2
\end{align}

and then

\begin{align}
\left \Vert P_\Omega^\perp (Z_t) \right \Vert_F^2
= \left \Vert Z_t \right \Vert_F^2
  - \left \Vert P_\Omega (Z_t) \right \Vert_F^2 \\
= \sum_{i = 1}^r \lambda_i^2 - \left \Vert Z_t \odot Y \right \Vert_F^2
\end{align}

Putting it all together we see

\begin{align}
||\tilde M^{(t)}||_F^2 
&= \left \Vert P_\Omega(M) \right \Vert_F^2 
  + \left \Vert P_\Omega^\perp \left(
      \sum_{i=1}^r \hat \lambda_i^{(t)} \hat U_i^{(t)} \hat V_i^{(t)^T}
    \right)
  \right \Vert_F^2 \\
&= \left \Vert M \right \Vert_F^2 + 
  \sum_{i = 1}^r \lambda_i^2 - 
  \left \Vert Z_t \odot Y \right \Vert_F^2
\end{align}

In code we will have a sparse matrix `M` and a list `s` with elements of the SVD. The first Frobenious norm is quick to calculate, but I am not sure how to calculate the other two frobenius norms.


```r
# s is a matrix defined in terms of it's svd
# G is a sparse matrix
# compute only elements of U %*% diag(d) %*% t(V) only on non-zero elements of G
# G and U %*% t(V) must have same dimensions

# maybe call this svd_perp?
svd_perp <- function(s, mask) {
  
  # note: must be dgTMatrix to get column indexes j larger
  # what if we used dlTMatrix here?
  m <- as(mask, "dgTMatrix")
  
  # the indices for which we want to compute the matrix multiplication
  # turn zero based indices into one based indices
  i <- m@i + 1
  j <- m@j + 1

  # gets rows and columns of U and V to multiply, then multiply
  ud <- s$u %*% diag(s$d)
  left <- ud[i, ]
  right <- s$v[j, ]

  # compute inner products to get elements of U %*% t(V)
  uv <- rowSums(left * right)

  # NOTE: specify dimensions just in case
  sparseMatrix(i = i, j = j, x = uv, dims = dim(mask))
}
```

Test it


```r
set.seed(17)

M <- rsparsematrix(8, 12, nnz = 30)
s <- svds(M, 5)

y <- as(M, "lgCMatrix")

Z <- s$u %*% diag(s$d) %*% t(s$v)

all.equal(
  svd_perp(s, M),
  Z * y
)
```

So, to take an eigendecomp you just need to be able to do $Mx$. To take an SVD, what do you need? matrix vector and matrix transpose vector multiplication


```r
set.seed(17)
r <- 5

M <- rsparsematrix(8, 12, nnz = 30)
y <- as(M, "lgCMatrix")

s <- svds(M, r)
Z <- s$u %*% diag(s$d) %*% t(s$v)

M_tilde <- M + Z * (1 - y)  # dense!

Z_perp <- svd_perp(s, M)
sum_singular_squared <- sum(M@x^2) + sum(s$d^2) - sum(Z_perp@x^2)

all.equal(
  sum(svd(M_tilde)$d^2),
  sum_singular_squared
)
```

### SVD of M tilde


```r
set.seed(17)
r <- 5

M <- rsparsematrix(8, 12, nnz = 30)
y <- as(M, "lgCMatrix")

s <- svds(M, r)
Z <- s$u %*% diag(s$d) %*% t(s$v)

M_tilde <- M + Z * (1 - y)  # dense!

svd_M_tilde <- svds(M_tilde, r)
svd_M_tilde
```



```r
Ax <- function(x, args) {
  drop(M_tilde %*% x)
}

Atx <- function(x, args) {
  drop(t(M_tilde) %*% x)
}

# is eigs_sym() with a two-sided multiply faster?
args <- list(u = s$u, d = s$d, v = s$v, m = M)
test1 <- svds(Ax, k = r, Atrans = Atx, dim = dim(M), args = args)

test1
svd_M_tilde

all.equal(
  svd_M_tilde,
  test1
)
```

So we're done our first sanity check of the function interface. Let $x$ be a vector. Now we want to calculate

\begin{align}
\tilde M^{(t)} x 
  &= \left[ P_\Omega(M) + P_\Omega^\perp (Z_t) \right] x \\
  &= \left[ P_\Omega(M) -
    P_\Omega (Z_t) +
    P_\Omega (Z_t) +
    P_\Omega^\perp (Z_t) \right] x \\
  &= P_\Omega(M - Z_t) x + Z_t x
\end{align}

where we can think of $R_t \equiv P_\Omega(M - Z_t)$ as "residuals" of sorts. Crucially, $R_t$ is sparse, and

\begin{align}
Z_t x &= (\hat U 
  \diag(\hat \lambda_1, ..., \hat \lambda_r)
  \hat V^t) x \\
  &= (\hat U 
  (\diag(\hat \lambda_1, ..., \hat \lambda_r)
  (\hat V^t x)))
\end{align}

So now the memory requirement of the computation has been reduced to that of two sparse matrix vector multiplications, rather than that of fitting the dense matrix $P_\Omega^\perp (Z_t)$ into memory.

Similarly, for the transpose, we have

\begin{align}
\tilde M^{{(t)}^T} x 
  &= \left[ P_\Omega(M) + P_\Omega^\perp (Z_t) \right]^T x \\
  &= \left[ P_\Omega(M) -
    P_\Omega (Z_t) +
    P_\Omega (Z_t) +
    P_\Omega^\perp (Z_t) \right]^T x \\
  &= P_\Omega(M - Z_t)^T x + Z_t^T x
\end{align}

This leads us to a second, less memory intensive implementation of `Ax()` and `Atx()`:


```r
# input: M, Z_t as a low-rank SVD list s

R <- M - svd_perp(s, M)  # residual matrix
args <- list(u = s$u, d = s$d, v = s$v, R = R)

Ax <- function(x, args) {
  drop(args$R %*% x + args$u %*% diag(args$d) %*% crossprod(args$v, x))
}

Atx <- function(x, args) {
  # TODO: can we use a crossprod() for the first multiplication here?
  drop(t(args$R) %*% x + args$v %*% diag(args$d) %*% crossprod(args$u, x))
}

# is eigs_sym() with a two-sided multiply faster?
test2 <- svds(Ax, k = r, Atrans = Atx, dim = dim(M), args = args)

all.equal(
  svd_M_tilde,
  test2
)
```


```r
relative_f_norm_change <- function(s_new, s) {
  # TODO: don't do the dense calculation here
  
  Z_new <- s_new$u %*% diag(s_new$d) %*% t(s_new$v)
  Z_new <- s$u %*% diag(s$d) %*% t(s$v)
  
  sum((Z_new - Z)^2) / sum(Z^2)
}
```



```r
low_rank_adaptive_impute <- function(M, r, epsilon = 1e-03) {

  # coerce M to sparse matrix such that we use sparse operations
  M <- as(M, "dgCMatrix")
  
  # low rank svd-like object, s ~ Z_1
  s <- low_rank_adaptive_initialize(M, r)  # line 1
  delta <- Inf
  d <- ncol(M)
  norm_M <- sum(M@x^2)

  while (delta > epsilon) {
    
    # update s: lines 4 and 5
    # take the SVD of M-tilde
    
    R <- M - svd_perp(s, M)  # residual matrix
    args <- list(u = s$u, d = s$d, v = s$v, R = R)
    
    s_new <- svds(Ax, k = r, Atrans = Atx, dim = dim(M), args = args)
    
    MtM <- norm_M + sum(s_new$d^2) - sum(svd_perp(s_new, M)^2)
    alpha <- (sum(MtM) - sum(s_new$d^2)) / (d - r)  # line 6
    
    s_new$d <- sqrt(s_new$d^2 - alpha)  # line 7
    
    # NOTE: skip explicit computation of line 8
    delta <- relative_f_norm_change(s_new, s)
    
    s <- s_new
    
    print(glue::glue("delta: {round(delta, 8)}, alpha: {round(alpha, 3)}"))
  }
  
  s
}
```


```r
out <- low_rank_adaptive_impute(M, r)
out
```


\begin{align}
\texttt{MtM} = \tilde M^{{(t)}^T} \tilde M^{(t)}
\end{align}


## Extension to a mixture of observed and unobserved missingness

originally solving an optimization vaguely of the form

\begin{align}
\left \Vert M - \hat M \right \Vert_F^2
\end{align}

where $M$ is a partially observed matrix. now, we let $M' = Y M$ where $Y$ is an indicator of whether or not $M$ was observed. Typically we have some setup like

\begin{align}
M = 
 \begin{bmatrix}
    \cdot & \cdot & 3     & 1     & \cdot \\
    3     & \cdot & \cdot & 8     & \cdot \\
    \cdot & -1    & \cdot & \cdot & \cdot \\
    \cdot & \cdot & \cdot & \cdot & \cdot \\
    \cdot & 2     & \cdot & \cdot & \cdot \\
    5     & \cdot & 7     & \cdot & 4
  \end{bmatrix}
, \qquad
Y = 
  \begin{bmatrix}
    \cdot & \cdot & 1     & 1     & \cdot \\
    1     & \cdot & \cdot & 1     & \cdot \\
    \cdot & 1     & \cdot & \cdot & \cdot \\
    \cdot & \cdot & \cdot & \cdot & \cdot \\
    \cdot & 1     & \cdot & \cdot & \cdot \\
    1     & \cdot & 1     & \cdot & 1
  \end{bmatrix}
\end{align}

Here the symbol $\cdot$ means that an entry of the matrix was unobserved.

but what we do now is, continuing to represent $M$ as a sparse matrix with no zero entries, is *observe* a bunch of zeros. So we might know that the upper triangule of $M$ has structurally missing zeros that we have observed are missing. These zeros are primarily important because they affect the residuals in our calculations. In this particular case, the take multiplication by $M$, a sparse operation, and make it into a dense operation.

At this point it becomes useful to introduce some additional notation. Let $\tilde \Omega$ be the set of indicies $(i, j)$ such that $M_{i, j}$ is non-zero. Observe that $\tilde \Omega \subset \Omega$. Then we have $P_{\tilde \Omega} (A) = P_\Omega (A)$.

\begin{align}
M = 
 \begin{bmatrix}
    0     & 0     & 3     & 1     & 0 \\
    3     & 0     & 0     & 8     & 0 \\
    \cdot & -1    & 0     & 0     & 0 \\
    \cdot & \cdot & \cdot & 0     & 0 \\
    \cdot & 2     & \cdot & \cdot & 0 \\
    5     & \cdot & 7     & \cdot & 4
  \end{bmatrix}
, \qquad
Y = 
  \begin{bmatrix}
    1     & 1     & 1     & 1     & 1 \\
    1     & 1     & 1     & 1     & 1 \\
    \cdot & 1     & 1     & 1     & 1 \\
    \cdot & \cdot & \cdot & 1     & 1 \\
    \cdot & 1     & \cdot & \cdot & 1 \\
    1     & \cdot & 1     & \cdot & 1
  \end{bmatrix}
, \qquad
M' = 
 \begin{bmatrix}
    \cdot & \cdot & 3     & 1     & \cdot \\
    3     & \cdot & \cdot & 8     & \cdot \\
    \cdot & -1    & \cdot & \cdot & \cdot \\
    \cdot & \cdot & \cdot & \cdot & \cdot \\
    \cdot & 2     & \cdot & \cdot & \cdot \\
    5     & \cdot & 7     & \cdot & 4
  \end{bmatrix}
\end{align}

And now we need to figure out how to calculate

\begin{align}
\tilde M^{(t)} x 
  &= \left[ P_\Omega(M) + P_\Omega^\perp (Z_t) \right] x \\
  &= \left[ P_\Omega(M) -
    P_\Omega (Z_t) +
    P_\Omega (Z_t) +
    P_\Omega^\perp (Z_t) \right] x \\
  &= P_\Omega(M)x  - P_\Omega(Z_t) x + Z_t x \\
  &= P_{\tilde \Omega}(M)x  - P_\Omega(Z_t) x + Z_t x
\end{align}

We already know how to calculate $P_{\tilde \Omega}(M)$ and $Z_t x$ using sparse operations, so we're left with $P_\Omega(Z_t) x$. Note that $P_\Omega(Z_t) x \neq P_{\tilde \Omega} (Z_t) x$ since $Z_t$ is not necessarily zero on $\tilde \Omega^\perp$. In other words $P_\Omega^\perp (Z_t) \neq Z_t$.

When $Y$ is dense, there is no way to avoid paying the computational cost of the dense or near dense computation. Our primary concern is fitting $Y$ into memory for large datasets. This is an issue when $Y$ is dense but with no discernible structure that permits a more compact representation.

Similarly we need to be able to calculate

\begin{align}
\tilde M^{(t)^T} x 
  &= \left[ P_\Omega(M) + P_\Omega^\perp (Z_t) \right]^T x \\
  &= \left[ P_\Omega(M)^T + P_\Omega^\perp (Z_t)^T \right] x \\
  &= \left[ P_\Omega(M)^T -
    P_\Omega (Z_t)^T +
    P_\Omega (Z_t)^T +
    P_\Omega^\perp (Z_t)^T \right] x \\
  &= P_\Omega(M)^T x  - P_\Omega(Z_t)^T x + Z_t^T x \\
  &= P_{\tilde \Omega}(M)^T x  - P_\Omega(Z_t)^T x + Z_t^T x
\end{align}

If we can fit $Y$ into memory, we can do a low-rank computation, only calculating elements $Z^{(t)}_{ij}$ when $Y_{ij} = 1$. When $Y$ is stored as a vector of row indices together with a vector of column indices (plus some information about the dimension), we can write the computation out:


```r
M <- Matrix(
  rbind(
    c(0, 0, 3, 1, 0),
    c(3, 0, 0, 8, 0),
    c(0, -1, 0, 0, 0),
    c(0, 0, 0, 0, 0),
    c(0, 2, 0, 0, 0),
    c(5, 0, 7, 0, 4)
  )
)

Y <- rbind(
  c(1, 1, 1, 1, 1),
  c(1, 1, 1, 1, 1),
  c(0, 1, 1, 1, 1),
  c(0, 0, 1, 1, 1),
  c(0, 1, 0, 1, 1),
  c(1, 0, 1, 0, 1)
)

s <- svds(M, 2)

Y <- as(Y, "TsparseMatrix")

# triplet form
# compressed column matrix form even better but don't
# understand the format
Y <- as(Y, "lgCMatrix")
Y <- as(Y, "lgTMatrix")

# ugh: RcppArmadillo only supports dgTMatrix rather than
# lgTMatrix which is somewhat unfortunate

# link: https://cran.r-project.org/web/packages/RcppArmadillo/vignettes/RcppArmadillo-sparseMatrix.pdf

Y

x <- rnorm(5)

# want to calculate
Z <- s$u %*% diag(s$d) %*% t(s$v)
out <- drop((Z * Y) %*% x)
out
```

At this point it's worth writing out explicitly how calculate $Z^{(t)}_{ij}$.

\begin{align}
Z^{(t)}_{ij} 
&= \left( \sum_{\ell=1}^r \hat U_\ell \hat d_\ell \hat V_\ell^T \right)_{ij}
\end{align}

For a visual reminder, this looks like (using $\diag(\hat d)$ and $\hat \Sigma$ somewhat interchangeably here)

\begin{align}
Z^{(t)}
= \hat U \diag(\hat d) \, \hat V^T
= 
\begin{bmatrix}
  U_1 & U_2 & \hdots & U_r \\
\end{bmatrix}
\begin{bmatrix}
  d_{1} & 0 & \hdots & 0 \\
  0 & d_{2} & \hdots & 0 \\
  \vdots & \vdots & \ddots & \vdots \\
  0 & 0 & \hdots & d_{r}
\end{bmatrix}
\begin{bmatrix}
  V_1^T \\
  V_2^T \\
  \vdots \\
  V_r^T
\end{bmatrix}
\end{align}

\begin{align}
\begin{bmatrix}
  U_{11} & U_{12} & \hdots & U_{1r} \\
  U_{21} & U_{22} & \hdots & U_{2r} \\
  \vdots & \vdots & \ddots & \vdots \\
  U_{n1} & U_{n2} & \hdots & U_{nr}
\end{bmatrix}
\begin{bmatrix}
  d_{1} & 0 & \hdots & 0 \\
  0 & d_{2} & \hdots & 0 \\
  \vdots & \vdots & \ddots & \vdots \\
  0 & 0 & \hdots & d_{r}
\end{bmatrix}
\begin{bmatrix}
  V_{11} & V_{21} & \hdots & V_{d1} \\
  V_{12} & V_{22} & \hdots & V_{d2} \\
  \vdots & \vdots & \ddots & \vdots \\
  V_{1r} & V_{2r} & \hdots & V_{dr}
\end{bmatrix}
\end{align}

n x d = (n x r) x (r x r) x (r x d)

(r x d) is after the transpose

\begin{align}
\begin{bmatrix}
  U_{11} & U_{12} & \hdots & U_{1r} \\
  U_{21} & U_{22} & \hdots & U_{2r} \\
  \vdots & \vdots & \ddots & \vdots \\
  U_{n1} & U_{n2} & \hdots & U_{nr}
\end{bmatrix}
\begin{bmatrix}
  d_{1} & 0 & \hdots & 0 \\
  0 & d_{2} & \hdots & 0 \\
  \vdots & \vdots & \ddots & \vdots \\
  0 & 0 & \hdots & d_{r}
\end{bmatrix}
\begin{bmatrix}
  V_{11} & V_{21} & \hdots & V_{d1} \\
  V_{12} & V_{22} & \hdots & V_{d2} \\
  \vdots & \vdots & \ddots & \vdots \\
  V_{1r} & V_{2r} & \hdots & V_{dr}
\end{bmatrix}
\end{align}

Also because I never remember anything, recall that $Z$ is $n \times d$, $x$ is $d \times 1$ and $Zx$ is $n \times 1$:

\begin{align}
(Zx)_i = \sum_{j=1}^d Z_{ij} \cdot x_j
\end{align}

(recall that $X_i$ always refers to the $i^{th}$ column of $X$ in this document, and $X_{i \cdot}$ refers to the $i^{th}$ row of $X$)

Also note that 

\begin{align}
Z_{ij} &= \sum_{k=1}^r U_{ik} d_k V_{kj} \\
&= \langle U_{i \cdot}, (D V^T)_j \rangle \\
&= \langle U_{i \cdot}, (D V)_{j \cdot} \rangle \\
&= U_{i \cdot}^T (D V)_{j \cdot}
\end{align}

Recall that $x \in \mathbb{R}^d$. Putting these together we find

\begin{align}
(Zx)_i &= \sum_{j=1}^d Z_{ij} \cdot x_j \\
&= \sum_{j=1}^d U_{i \cdot}^T (D V)_{j \cdot} \cdot x_j
\end{align}


```r
# mask as a pair list
# L and Z / svd are both n x d matrices
# x is a d x 1 matrix / vector
masked_svd_times_x <- function(s, mask, x) {
  
  stopifnot(inherits(mask, "lgTMatrix"))
  
  u <- s$u
  d <- s$d
  v <- s$v
  
  zx <- numeric(nrow(u))
  
  # lgTMatrix uses zero based indexing, add one
  row <- mask@i + 1
  col <- mask@j + 1
  
  # need to loop over index of indexes
  # double looping over i and j here feels intuitive
  # but is incorrect
  for (idx in seq_along(row)) {
    i <- row[idx]
    j <- col[idx]
    
    z_ij <- sum(u[i, ] * d * v[j, ])
    zx[i] <- zx[i] + x[j] * z_ij
  }
  
  zx
}

# how to calculate just one element of the reconstructed
# data using the SVD

i <- 6
j <- 4

sum(s$u[i, ] * s$d * s$v[j, ])
Z[i, j]

# the whole masked matrix multiply

Z <- s$u %*% diag(s$d) %*% t(s$v)
out <- drop((Z * Y) %*% x)

# check that we did this right
all.equal(
  masked_svd_times_x(s, Y, x),
  out
)
```

This is gonna be painfully slow in R so we rewrite in C++


```cpp
#include <RcppArmadillo.h>

using namespace arma;

// [[Rcpp::depends(RcppArmadillo)]]
// [[Rcpp::export]]

vec masked_svd_times_x_impl(
  const mat& U,
  const rowvec& d,
  const mat& V,
  const vec& row,
  const vec& col,
  const vec& x) {
  
  int i, j;
  double z_ij;
  
  vec zx = zeros<vec>(U.n_rows);
  
  for (int idx = 0; idx < row.n_elem; idx++) {
    
    i = row(idx);
    j = col(idx);
    
    // % does elementwise multiplication in Armadillo
    // accu() gives the sum of elements of resulting vector
    z_ij = accu(U.row(i) % d % V.row(j));
    
    zx(i) += x(j) * z_ij;
  }
  
  return zx;
}
```


```r
# wrap with slightly nicer interface
masked_svd_times_x_cpp <- function(s, mask, x) {
  drop(masked_svd_times_x_impl(s$u, s$d, s$v, mask@i, mask@j, x))
}
```


```r
bench::mark(
  masked_svd_times_x_cpp(s, Y, x),
  masked_svd_times_x(s, Y, x)
)
```

Now we also need to work out $(Z_t^T x)_i$, which I am hella blanking on.



```r
library(Matrix)
library(RSpectra)
library(testthat)

set.seed(27)

# create a random 8 x 12 sparse matrix with 30 nonzero entries
M <- rsparsematrix(8, 11, nnz = 20)
M

s <- svds(M, 3)

Y <- M != 0
mask <- as(M, "TsparseMatrix")

u <- s$u
d <- s$d
v <- s$v


Z <- u %*% diag(d) %*% t(v)

Zt2 <- v %*% diag(d) %*% t(u)

expect_equal(Zt2, t(Z))

x <- rnorm(8)
Ztx <- t(Z * Y) %*% x

ztx <- numeric(nrow(v))

# lgTMatrix uses zero based indexing, add one
row <- mask@i + 1
col <- mask@j + 1

# need to loop over index of indexes
# double looping over i and j here feels intuitive
# but is incorrect
for (idx in seq_along(row)) {
  
  # here's where we do the row / column swap
  i <- row[idx]
  j <- col[idx]
  
  z_ij <- sum(u[i, ] * d * v[j, ])
  ztx[j] <- ztx[j] + x[i] * z_ij
}

Ztx
ztx

impl_result <- p_omega_ztx_impl(u, d, v, mask@i, mask@j, x)

testthat::expect_equivalent(
  drop(ztx),
  drop(Ztx)
)
```



```r
# add the upper triangle back to Y since we're in the citation
# setting
Y <- Y | upper.tri(Y)

obs_Zt <- t(Z * Y)                 # project SVD onto observed matrix

Y <- as(Y, "TsparseMatrix")
impl_result <- p_omega_ztx_impl(s$u, s$d, s$v, Y@i, Y@j, x)

expect_equal(
  drop(impl_result),
  drop(obs_Zt %*% x)
)
```

START MISC

$Z$ is an $n \times d$ matrix. Thus we must have $x \in \mathbb{R}^n$, and then $Z^T x \in \mathbb{R}^d$.

\begin{align}
(Z^Tx)_i = \sum_{j=1}^n Z^T_{ij} \cdot x_j
\end{align}

Also note that 

\begin{align}
Z^T_{ij} &= Z_{ji} \\
&= \sum_{k=1}^r d_k U_{jk} V_{ki}^T
\end{align}

Putting these together we find

\begin{align}
(Z^T x)_i &= \sum_{j=1}^n Z^T_{ij} \cdot x_j \\
&= \sum_{j=1}^n \sum_{k=1}^r d_k U_{jk} V_{ki} \cdot x_j \\
&= \sum_{j=1}^n U_{j \cdot}^T (D V)_{i \cdot} \cdot x_j
\end{align}

END MISC

TODO:

- the tranpose multiplication
- the average singular value calculation

Recall that to calculate the average singular value we want

\begin{align}
||\tilde M^{(t)}||_F^2
&= \left \Vert M \right \Vert_F^2 + 
  \sum_{i = 1}^r \lambda_i^2 - 
  \left \Vert Z_t \odot Y \right \Vert_F^2
\end{align}

KEY NOTE: $\lambda_i$ is the $i^{th}$ singular value of the $Z^{(t-1)}$, *not* $Z^{(t)}$.

This is the other computationally intensive part so let's write it up in Armadillo as well


```cpp
#include <RcppArmadillo.h>

using namespace arma;

// [[Rcpp::depends(RcppArmadillo)]]
// [[Rcpp::export]]

double p_omega_f_norm_impl(
    const mat& U,
    const rowvec& d,
    const mat& V,
    const vec& row,
    const vec& col) {
  
  int i, j;
  double total = 0;
  
  for (int idx = 0; idx < row.n_elem; idx++) {
    
    i = row(idx);
    j = col(idx);
    
    total += accu(U.row(i) % d % V.row(j));
  }
  
  return total;
}
```


```r
# wrap with slightly nicer interface
p_omega_f_norm_cpp <- function(s, mask) {
  p_omega_f_norm_impl(s$u, s$d, s$v, mask@i, mask@j)
}
```


```r
all.equal(
  sum(Z * Y),
  p_omega_f_norm_cpp(s, Y)
)
```

### Citation graphs

Now it turns out that if we have a generally sparse mask of additional zeros, what we've done so far works out pretty nicely. When we have lots and lots of additional zeros, then we end up redoing the same computations over and over when we expand $Z^{(t)}$ in the $\tilde M^{(t)}x$ multiplication steps.

We're particularly interested in the case where $M$ is a sparse, but we know that all the elements of the upper triangle are observed. This is the case for the adjacency matrix of a citation network, for example.

The fundamental issue now is how to reduce the *time complexity* (recall that we previously reduced the space complexity) of the $P_\Omega(Z_t) x$ operation in

\begin{align}
\tilde M^{(t)} x = P_{\tilde \Omega}(M)x  - P_\Omega(Z_t) x + Z_t x
\end{align}

Let $U$ be the set of pairs $(i, j)$ such that $i > j$. Note that this inequality is strict -- in citation matrices we will assume items do not cite themselves (*loud coughing*). Next let $\tilde U \subset \Omega$ be the set of pairs $(i, j)$ where $i \le j$. Observe that $U \cup \tilde U = \Omega$. So we can write

\begin{align} 
P_\Omega(Z_t) \, x = P_U (Z_t) \, x + P_{\tilde U} (Z_t) \, x
\end{align}

$P_{\tilde U} (Z_t) x$ is again easy so we focus on:

\begin{align}
\left( P_U (Z_t) \, x \right)_i
&= \sum_{j=1}^d Z_{ij} \cdot \mathbf 1 (i < j) \cdot x_j  \\
&= \sum_{j=1}^d \left( \sum_{k=1}^r U_{ik} (DV)_{kj} \right) \cdot x_j \cdot \mathbf 1 (i < j) \\
&= \sum_{j=1}^d \left( \sum_{k=1}^r U_{ik} (DV)_{kj} x_j \right) \cdot \mathbf 1 (i < j)
\end{align}

Now let $W$ be an $r \times d$ matrix such that that $j^{th}$ column of $W$ is $(DV)_j \cdot x_j$. Then

\begin{align}
...
&= \sum_{j=1}^d \sum_{k=1}^r U_{ik} (DV)_{kj} x_j \cdot \mathbf 1 (i < j) \\
&= \sum_{k=1}^r \sum_{j=1}^d U_{ik} (DV)_{kj} x_j \cdot \mathbf 1 (i < j) \\
&= \sum_{k=1}^r U_{ik} \sum_{j=1}^d (DV)_{kj} x_j \cdot \mathbf 1 (i < j) \\
&= \sum_{i=1}^n \langle u_j, v_i \rangle x_i \mathbf 1 (i < j) \\
&= \sum_{i=1}^n \sum_{\ell = 1}^r u_{j \ell} v_{i \ell} 
  x_i \mathbf 1 (i < j) \\
&= \sum_{\ell = 1}^r u_{j \ell} \sum_{i=1}^j v_{i \ell} x_i
\end{align}


Also because I never remember anything, recall that $Z$ is $n \times d$, $x$ is $d \times 1$ and $Zx$ is $n \times 1$:

\begin{align}
(Zx)_i = \sum_{j=1}^d Z_{ij} \cdot x_j
\end{align}

(recall that $X_i$ always refers to the $i^{th}$ column of $X$ in this document, and $X_{i \cdot}$ refers to the $i^{th}$ row of $X$)

Also note that 

\begin{align}
Z_{ij} &= \sum_{k=1}^r U_{ik} d_k V_{kj} \\
&= \langle U_{i \cdot}, (D V^T)_j \rangle \\
&= \langle U_{i \cdot}, (D V)_{j \cdot} \rangle \\
&= U_{i \cdot}^T (D V)_{j \cdot}
\end{align}

Recall that $x \in \mathbb{R}^d$. Putting these together we find

\begin{align}
(Zx)_i &= \sum_{j=1}^d Z_{ij} \cdot x_j \\
&= \sum_{j=1}^d U_{i \cdot}^T (D V)_{j \cdot} \cdot x_j
\end{align}


```r
library(Matrix)
library(RSpectra)
library(testthat)

set.seed(27)

M <- rsparsematrix(8, 11, nnz = 20)
M

n <- nrow(M)  # 8
d <- ncol(M)  # 11

x <- rnorm(d)

Y <- as(upper.tri(M), "CsparseMatrix")

s <- svds(M, 3)

U <- s$u
DVt <- diag(s$d) %*% t(s$v)

expected <- drop((U %*% DVt * Y) %*% x)
expected

# TODO: be careful how broadcasting works
# could broadcast on either rows or columns here
# especially in square case
DVtx <- DVt * x

CDVx <- matrixStats::colCumsums(DVx)

out <- numeric(n)

for (j in 1:n) {
  out[j] <- sum(U[j, ] * CDVx[, j])
}

expect_equal(
  drop(out),
  drop(expected)
)
```

Okay so I'm getting stuck making this work for citation matrices so let's just do a standard matrix multiply this way


```r
library(Matrix)
library(RSpectra)
library(testthat)

set.seed(27)

M <- rsparsematrix(8, 11, nnz = 20)
M

n <- nrow(M)  # 8
d <- ncol(M)  # 11

x <- rnorm(d)

s <- svds(M, 3)

U <- s$u
DVt <- diag(s$d) %*% t(s$v)

expected <- drop((U %*% DVt) %*% x)
expected

# TODO: be careful how broadcasting works
# could broadcast on either rows or columns here
# especially in square case
DVtx <- DVt * x
DVtx

U %*% DVt %*% x

out <- numeric(n)

for (j in 1:n) {
  out[j] <- sum(U[j, ] * DVt[, j]) * x[j]
}

out <- U %*% DVt %*% x

expect_equal(
  drop(out),
  drop(expected)
)
```



Put $CVX = \sum_{i=1}^j v_{i \ell} x_i$

### A naive solution: the epsilon trick

issue: we've moved back into dense computation land

## References