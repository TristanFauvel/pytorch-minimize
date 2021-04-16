# PyTorch Minimize

Pytorch-minimize represents a collection of utilities for minimizing scalar functions of one or more variables in PyTorch. 
It is inspired heavily by SciPy's `optimize` module and MATLAB's [Optimization Toolbox](https://www.mathworks.com/products/optimization.html). 
Unlike SciPy and MATLAB, which use numerical approximations to function derivatives, pytorch-minimize uses _real_ first- and second-order derivatives at all times, computed seamlessly behind the scenes with autograd.
Both CPU and CUDA are supported.

Author: Reuben Feinman

__At a glance:__

```python
import torch
from fmin import minimize

def rosen(x):
    return torch.sum(100*(x[..., 1:] - x[..., :-1]**2)**2 
                     + (1 - x[..., :-1])**2)

# initial point
x0 = torch.tensor([1., 8.])

# BFGS
result = minimize(rosen, x0, method='bfgs')

# Newton Conjugate Gradient
result = minimize(rosen, x0, method='newton-cg')

# Newton Exact
result = minimize(rosen, x0, method='newton-exact')
```

__Solvers:__ BFGS, L-BFGS, Newton Conjugate Gradient (CG), Newton Exact

__Examples:__ See the [Rosenbrock minimization notebook](https://github.com/rfeinman/pytorch-minimize/blob/master/examples/rosen_minimize.ipynb) for a demonstration of function minimization with a handful of different algorithms.

## Motivation
Although PyTorch offers many routines for stochastic optimization, utilities for deterministic optimization are scarce; only L-BFGS is included in the `optim` package, and it's modified for mini-batch training.

MATLAB and SciPy are industry standards for deterministic optimization. 
These libraries have a comprehensive set of routines; however, automatic differentiation is not supported.* 
Therefore, the user must specify 1st- and 2nd-order gradients explicitly (if they are known) or use finite-difference approximations.

The motivation for this library is to offer a set of tools for deterministic optimization with analytical gradients via PyTorch's autograd.

_

*MATLAB offers minimal autograd support via the Deep Learning Toolbox, but the integration is not seamless: data must be converted to "dlarray" structures, and only a [subset of functions](https://www.mathworks.com/help/deeplearning/ug/list-of-functions-with-dlarray-support.html) are supported.
Furthermore, derivatives must still be constructed and provided as function handles. 
Pytorch-minimize uses autograd to compute derivatives behind the scenes, so all you ever need to provide is your function.

## Library

The pytorch-minimize library includes solvers for general-purpose function minimization (unconstrained & constrained), as well as for nonlinear least squares problems.

### Unconstrained Minimizers

1. __BFGS/L-BFGS.__ BFGS is a cannonical quasi-Newton method for unconstrained optimization. I've implemented both the standard BFGS and the "limited memory" L-BFGS. For smaller scale problems where memory is not a concern, BFGS should be significantly faster than L-BFGS (especially on CUDA) since it avoids Python for loops and instead uses pure torch.
   
2. __Newton Conjugate Gradient (CG).__ The Newton-Raphson method is a staple of unconstrained optimization. Although computing full Hessian matrices with PyTorch's reverse-mode automatic differentiation can be costly, computing Hessian-vector products is cheap, and it also saves a lot of memory. The Conjugate Gradient (CG) variant of Newton's method is an effective solution for unconstrained minimization with Hessian-vector products. I've implemented a lightweight NewtonCG minimizer that uses HVP for the linear inverse sub-problems.

3. __Newton Exact.__ In some cases, we may prefer a more precise variant of the Newton-Raphson method at the cost of additional complexity. I've also implemented an "exact" variant of Newton's method that computes the full Hessian matrix and uses Cholesky factorization for linear inverse sub-problems. When Cholesky fails--i.e. the Hessian is not positive definite--the solver resorts to one of two options as specified by the user: 1) steepest descent direction (default), or 2) solve the inverse hessian with LU factorization.

### Constrained Minimizers

1. __Trust-Region Constrained Algorithm.__ Pytorch-minimize includes a single constrained minimization routine based on SciPy's `trust-constr` method. The algorithm accepts generalized nonlinear constraints and variable boundries via the "constr" and "bounds" arguments. For equality constrained problems, it is an implementation of the Byrd-Omojokun Trust-Region SQP method. When inequality constraints are imposed, the trust-region interior point method is used. 

NOTE: The current trust-region constrained minimizer is not a custom implementation, but rather a wrapper for SciPy's `optimize.minimize` routine (all rights reserved). It uses autograd behind the scenes to build jacobian & hessian callables before invoking scipy. Inputs and objectivs should use torch tensors like other pytorch-minimize routines. CUDA is supported but not recommended; data will be moved back-and-forth between GPU/CPU. This minimizer is not currently accessible through the `minimize` routine and instead must be imported directly as follows:

    from fmin import fmin_trust_constr

### Nonlinear Least Squares

The library also includes specialized solvers for nonlinear least squares problems. These solvers revolve around the Gauss-Newton method, a modification of Newton's method tailored to the lstsq setting. The least squares interface can be imported as follows:

    from fmin import least_squares

The least squares solver is heavily motivated by scipy's `optimize.least_squares`. Much of the scipy code was borrowed directly (all rights reserved) and ported from numpy to torch. Rather than having the user provide a jacobian function, in the new interface jacobian-vector products are computed seamlessly on the backend with autograd. At the moment, only the Trust Region Reflective ("trf") method is implemented, and bounds are not yet supported.

## Examples

The [Rosenbrock minimization tutorial](https://github.com/rfeinman/pytorch-minimize/blob/master/examples/rosen_minimize.ipynb) demonstrates how to use pytorch-minimize to find the minimum of a scalar-valued function of multiple variables using various optimization strategies.

In addition, the [SciPy benchmark](https://github.com/rfeinman/pytorch-minimize/blob/master/examples/scipy_benchmark.py) provides a comparison of pytorch-minimize solvers to their analogous solvers from the `scipy.optimize` library. 
For those transitioning from scipy, this script will help get a feel for the design of the current library. 
Unlike scipy, jacobian and hessian functions need not be provided to pytorch-minimize solvers, and numerical approximations are never used.

For constrained optimization, the [adversarial examples tutorial](https://github.com/rfeinman/pytorch-minimize/blob/master/examples/constrained_optimization_adversarial_examples.ipynb) demonstrates how to use the trust-region constrained routine to generate an optimal adversarial perturbation given a constraint on the perturbation norm.

## Onging work

- Optimizer API. Coming soon there will be a new alternative API revolving around the `torch.optim.Optimizer` class. An early prototype can be found at `fmin/optim.py`. It has not been rigorously tested, and documentation is limited.
- Custom constrained optimizers.
