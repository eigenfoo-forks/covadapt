# Better mass matrices for NUTS

This is an experimental implementation of a low rank approximation of
mass matrices for hamiltonian MCMC samplers, specifically for pymc3.

*This is for experimentation only! Do not use for actual work (yet)!*

But feel welcome to try it out, and tell me how it worked for your models!

## Install

Make sure you have pymc3 and numba installed. Then do

```
pip install git+https://github.com/aseyboldt/covadapt.git
```

## Usage

To use the new adaptation, define a pymc3 model as usual, and then create
a potential from this library and pass it to `pm.NUTS`.

```python
import covadapt
import pymc3 as pm

with pm.Model() as model:
    # Some pymc3 model
    pm.Normal('y', shape=100)

    # Define the potential
    pot = covadapt.EigvalsAdapt(
        model.ndim,
        np.zeros(model.ndim),
        estimators=[
            lambda samples, grads:
                covadapt.eigh_lw_samples_grads(
                    samples, grads, n_eigs=20, n_eigs_grad=20, n_final=40
                )
        ],
        display=True,
    )

    # Initialize the NUTS sampler with the new potential
    step = pm.NUTS(potential=pot)
    trace = pm.sample(step=step, draws=1000, tune=2000, chains=4)
```

And a complete example that fails with the pymc3 standard sampler:
```python
n = 500

U = np.array([[1, 0, -3, 0, 0, 6] + [0] * (n - 6),
              [0, 5, 0, 3, -2, 0] + [0] * (n - 6)]).T

U = U / np.sqrt((U ** 2).sum(0))[None, :]
true_eigvals = U
Σ = np.diag([2000000, 0.00001])
cov = U @ Σ @ U.T + (np.eye(n) - U @ U.T)


with pm.Model() as model:
    pm.MvNormal('a', shape=n, mu=0, cov=cov)

    pot = covadapt.EigvalsAdapt(
        model.ndim,
        np.zeros(model.ndim),
        estimators=[
            lambda samples, grads:
                covadapt.eigh_lw_samples_grads(
                    samples, grads, n_eigs=20, n_eigs_grad=20, n_final=40
                )
        ],
        display=True,
    )

    step = pm.NUTS(potential=pot)
    trace = pm.sample(step=step, draws=1000, tune=2000, chains=4)
```

## How it works

Given some eigenvectors Q and eigenvalues Σ, we can represent a covariance
matrix $C = I + QΣQ^T - QQ^T$ without storing anything more than those
few vectors. The resulting matrix has the given eigenvectors and values,
all other eigenvalues are 1. In order to run NUTS or some othe HMC we need
matrix vector products $Cv$ and $C^{-\tfrac{1}{2}}v$, where $C^{-\tfrac{1}{2}}$
is some factorization of $C^{-1}$. Thanks to the structure of $C$ we can
implement both matrix actions easily.

We also need some estimates for eigenvectors of the posterior covariance.
One way to get some is to use the ledoit-wolf estimate of some samples.
We do not want to store the whole matrix, but we can estimate the
shrinkage parameter in batches (see the implementation in sklearn),
and given the shrinkage parameter we can implement the matrix action
and use lanczos or something similar to get large eigenvalues of that.
This is what `covadapt.eigvals_lw.eigh_lw_samples` does.

Interestingly, if you have a gaussian posterior, and you look at the gradients
of the logp at the points of the samples, the covariance of those gradients
will be the inverse posterior covariance. (This is because gradients are
covariant while the values are contravariant, and for the standard normal both
inner products are the identity). So we can do the same we did with the
samples, but with the gradients at the positions of the samples.  This will
give use estimates of the *small* eigenvalues of the posterior covariance.
Unfortunatly, the two sets of eigenvectors are not orthogonal.  I take the mean
of the two estimates on the matrix-log scale and estimate small and large
eigenvectors of that mean.  This is `covadapt.eigvals_lw.eigh_regularized_grad`.

## Acknowledgment

A lot of the work that went into this package was during my time at Quantopian,
while trying to improve sampling of a (pretty awesome) model for portfolio
optimization. Thanks a lot for making that possible!

![Quantopian logo](https://raw.githubusercontent.com/pymc-devs/pymc3/master/docs/quantopianlogo.jpg)


## Some random rambling

The third option is trying to use a different way to regularize eigenvector
estimates: We can define the eigenvector of a matrix as $\argmax_{|x| = 1}
x^TCx$.  We can modify this to include some regularization:

$$
\argmax_{|x| = 1} x^Tx - \gamma |x|_1.
$$

Unfortunately this introduces some problems, as the objective is not convex,
(maybe it is spherically convex? eg Ferreira, Orizon & Iusem, Alfredo & Zoltán
Németh, Sándor. (2014). Concepts and techniques of Optimization on the sphere.
Top. accepted.) and the loss is not differentiable at $x_i = 0$. There is quite
a lot of literature about optimization with l1 loss, so maybe this would work
out better with a bit of work. The parameter gamma could maybe be estimated
using cross validation of the samples.

The current `covadapt.eigvals_reg` code uses this approch.

Alternatively we could also try to do the whole regularized PCA in one go,
and optimize something like: Given posterior samples $X$, find $Q$ orthogonal,
$D$ diagonal and $\Sigma$ such that
$$
\text{std_normal_logp}(C^{-\tfrac{1}{2}}X) - \gamma |Q|_1
$$
is minimal, where $C^{-\tfrac{1}{2}} = (I + Q^T\Sigma^{-1/2}Q - Q^TQ)D^{-1/2}$.

Paper about sparse eigenvectors: https://arxiv.org/pdf/1408.6686.pdf
