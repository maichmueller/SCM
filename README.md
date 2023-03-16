| OS        |  Status |
| :-------------: |:-------------:|
| Linux       | ![L Py 3.7 - 3.9](https://github.com/maichmueller/scm/workflows/L%20Py%203.7%20-%203.9/badge.svg)    |
| Windows | ![W Py 3.7 - 3.9](https://github.com/maichmueller/scm/workflows/W%20Py%203.7%20-%203.9/badge.svg) |
| Mac | ![M Py 3.7 - 3.9](https://github.com/maichmueller/scm/workflows/M%20Py%203.7%20-%203.9/badge.svg) |

A Python package implementing Structural Causal Models (SCM).

The library uses the CAS library [SymPy](https://github.com/sympy/sympy) to allow the user to state arbitrary assignment functions and noise distributions as supported by SymPy and builds the DAG with [networkx](https://github.com/networkx/networkx).

It supports the features:
  - Sampling
  - Intervening
  - Plotting
  - Printing

 and by extension all methods on a DAG provided by networkx after accessing the member variable dag

# Installation
Either install via pip
```
pip install scmodels
```
or via cloning the repository and running the setup.py file
```
git clone https://github.com/maichmueller/scm
cd scm
python setup.py install
```

# Building an SCM

In order to declare the SCM $X \rightarrow Y \leftarrow Z \rightarrow X$

with the assignments

$Z \sim \text{LogLogistic} (\alpha=1,\beta=1)$

$X=3Z^2 \cdot N \quad [N = \text{LogNormal}(\mu=1,\sigma=1)]$

$Y = 2Z + \sqrt{X} + N \quad [N=\text{Normal}(\mu=2,\sigma=1)]$

there are 2 different ways implemented.

## 1. List Of Strings
Describe the assignments as strings of the form:

'VAR = FUNC(Noise, parent1, parent2, ...), Noise ~ DistributionXYZ'

Note that - out of convenience - in this case, one does not need to (and isn't allowed to)
restate the noise symbol string in the distribution (as would otherwise be necessary
in constructing sympy distributions).


```python
from scmodels import SCM

assignment_seq = [
    "Z = M, M ~ LogLogistic(alpha=1, beta=1)",
    "X = N * 3 * Z ** 2, N ~ LogNormal(mean=1, std=1)",
    "Y = P + 2 * Z + sqrt(X), P ~ Normal(mean=2, std=1)",
    "V = N**P + X, N ~ Normal(0,1) / P ~ Bernoulli(0.5)",
    "W = exp(T) - log(M) * N + Y, M ~ Exponential(1) / T ~ StudentT(0.5) / N ~ Normal(0, 2)",
]

myscm = SCM(assignment_seq)
```

Agreements:
- The name of the noise variable in the distribution specification
(e.g. `P ~ Normal(mean=2, std=1)`) has to align with the noise variable
name (`P`) of the assignment string.
- Multiple noise models can be composited as done for variable `W` in the above example.
The noise model string segment must specify all individual noise models separated by a '/'

## 2. Assignment Map

One can construct the SCM via an assignment map with the variables as keys
and a tuple defining the assignment and the noise.

### 2-Tuple: Assignments via SymPy parsing

To refer to SymPy's string parsing capability (this includes numpy functions) provide a dict entry
with a 2-tuple as value of the form:

`'var': ('assignment string', noise)`




```python
from sympy.stats import LogLogistic, LogNormal, Normal, Bernoulli

assignment_map = {
    "Z": (
        "M",
        LogLogistic("M", alpha=1, beta=1)
    ),
    "X": (
        "N * 3 * Z ** 2",
        LogNormal("N", mean=1, std=1),
    ),
    "Y": (
        "P + 2 * Z + sqrt(X)",
        Normal("P", mean=2, std=1),
    ),
    "V": (
        "N**P + X",
        [Normal("N", 0, 1), Bernoulli("P", 0.5)]
    )
}

myscm2 = SCM(assignment_map)
```

Agreements:
- the name of the noise distribution provided in its constructor
(e.g. `Normal("N", mean=2, std=1)`) has to align with the noise variable
name (`N`) of the assignment string.
- Multiple noise models in the assignment must be wrapped in an iterable (e.g. a list `[]`, or tuple `()`)

### 3-Tuple: Assignments with arbitrary callables

One can also declare the SCM via specifying the variable assignment in a dictionary with the
variables as keys and as values a sequence of length 3 of the form:

`'var': (['parent1', 'parent2', ...], Callable, Noise)`

This allows the user to supply complex functions outside the space of analytical functions.


```python
import numpy as np


def Y_assignment(p, z, x):
    return p + 2 * z + np.sqrt(x)


functional_map = {
    "Z": (
        [],
        lambda m: m,
        LogLogistic("M", alpha=1, beta=1)
    ),
    "X": (
        ["Z"],
        lambda n, z: n * 3 * z ** 2,
        LogNormal("N", mean=1, std=1),
    ),
    "Y": (
        ["Z", "X"],
        Y_assignment,
        Normal("P", mean=2, std=1),
    ),
}

myscm3 = SCM(functional_map)
```

Agreements:
- The callable's first parameter MUST be the noise input (unless the noise distribution is `None`).
- The order of variables in the parents list determines the semantic order of input for parameters in the functional
(left to right).

# Features

## Prettyprint

The SCM supports a form of informative printing of its current setup,
which includes mentioning active interventions and the assignments.


```python
print(myscm)
```

    Structural Causal Model of 5 variables: Z, X, Y, V, W
    Variables with active interventions: []
    Assignments:
    Z := f(M) = M	 [ M ~ LogLogistic(alpha=1, beta=1) ]
    X := f(N, Z) = N * 3 * Z ** 2	 [ N ~ LogNormal(mean=1, std=1) ]
    Y := f(P, Z, X) = P + 2 * Z + sqrt(X)	 [ P ~ Normal(mean=2, std=1) ]
    V := f(N, P, X) = N**P + X	 [ N ~ Normal(mean=0, std=1), P ~ Bernoulli(p=0.5, succ=1, fail=0) ]
    W := f(M, T, N, Y) = exp(T) - log(M) * N + Y	 [ M ~ Exponential(rate=1), T ~ StudentT(nu=0.5), N ~ Normal(mean=0, std=2) ]


In the case of custom callable assignments, the output is less informative


```python
print(myscm3)
```

    Structural Causal Model of 3 variables: Z, X, Y
    Variables with active interventions: []
    Assignments:
    Z := f(M) = __unknown__	 [ M ~ LogLogistic(alpha=1, beta=1) ]
    X := f(N, Z) = __unknown__	 [ N ~ LogNormal(mean=1, std=1) ]
    Y := f(P, Z, X) = __unknown__	 [ P ~ Normal(mean=2, std=1) ]


## Interventions
One can easily perform interventions on the variables,
e.g. a Do-intervention or also general interventions, which remodel the connections, assignments, and noise distributions.
For general interventions, the passing structure is dict of the following form:

    {var: (New Parents (Optional), New Assignment (optional), New Noise (optional))}

Any part of the original variable state, that is meant to be left unchanged, has to be passed as `None`.
E.g. to assign a new callable assignment to variable `X` without changing parents or noise, one would call:


```python
my_new_callable = lambda n, z: n + z

myscm.intervention({"X": (None, my_new_callable, None)})
```

For the example of the do-intervention $\text{do}(X=1)$,
one can use the helper method `do_intervention`. The pendant for noise interventions is called `soft_intervention`:


```python
myscm.do_intervention([("X", 1)])

from sympy.stats import FiniteRV

myscm.soft_intervention([("X", FiniteRV(str(myscm["X"].noise), density={-1: .5, 1: .5}))])
```

Calling `undo_intervention` restores the original state of all variables from construction time, that have been passed. One can optionally specify,
If no variables are specified (`variables=None`), all interventions are undone.


```python
myscm.undo_intervention(variables=["X"])
```

## Sampling

The SCM allows drawing as many samples as needed through the method `myscm.sample(n)`.


```python
n = 5
samples = myscm.sample(n)

display_data(samples)
```


|    |         Z |          X |        Y |         V |            W |
|----|-----------|------------|----------|-----------|--------------|
|  0 | 2.95064   | 38.7617    | 13.9738  | 38.835    | 14.2153      |
|  1 | 0.327761  |  1.46947   |  5.88667 |  2.46947  |  5.01149     |
|  2 | 0.0603847 |  0.0807284 |  3.77758 |  1.08073  |  7.2844      |
|  3 | 0.148583  |  0.908464  |  4.12864 |  0.307823 |  2.28556e+07 |
|  4 | 0.172203  |  0.335466  |  2.06417 |  1.33547  |  0.158294    |


If infinite sampling is desired, one can also receive a sampling generator through


```python
container = {var: [] for var in myscm}
sampler = myscm.sample_iter(container)
```

`container` is an optional target dictionary to store the computed samples in.


```python
import pandas as pd

for i in range(n):
    next(sampler)

display_data(container)
```


|    |         Z |        X |        Y |       V |          W |
|----|-----------|----------|----------|---------|------------|
|  0 | 0.389587  | 0.795743 | 0.996379 | 1.79574 | 95.7612    |
|  1 | 0.0947226 | 0.204903 | 2.00336  | 1.95693 |  5.17215   |
|  2 | 1.41253   | 6.10847  | 8.60707  | 7.10847 | 98.7214    |
|  3 | 0.522453  | 1.35502  | 4.32012  | 2.35502 |  0.0663619 |
|  4 | 0.509057  | 2.3294   | 4.47076  | 2.06578 |  5.43165   |


If the target container is not provided, the generator returns a new `dict` for every sample.


```python
sample = next(myscm.sample_iter())

display_data(sample)
```


|    |       Z |       X |       Y |       V |       W |
|----|---------|---------|---------|---------|---------|
|  0 | 1.74147 | 11.3284 | 8.80388 | 12.3284 | 9.69305 |


## Plotting
If you have graphviz installed, you can plot the DAG by calling

```python
myscm.plot(node_size=1000, alpha=1)
```

![example_plot](https://github.com/maichmueller/scm/blob/master/docs/images/example_plot.png)
