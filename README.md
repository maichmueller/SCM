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

To build the DAG

![X \rightarrow Y \leftarrow Z \rightarrow X](https://latex.codecogs.com/svg.latex?&space;X{\rightarrow}{Y}{\leftarrow}{Z}{\rightarrow}X)


with the assignments

![Z ~ LogLogistic(alpha=1, beta=1)](https://latex.codecogs.com/svg.latex?&space;Z\sim\text{LogLogistic}(\alpha=1,\beta=1)])

![X = 3Z^2{\cdot}N](https://latex.codecogs.com/svg.latex?&space;X={3Z^2}{\cdot}N\quad[N=\text{LogNormal}(\mu=1,\sigma=1)])

![Y = 2Z + \sqrt{X} + N](https://latex.codecogs.com/svg.latex?&space;Y=2Z+\sqrt{X}+N\quad[N=\text{Normal}(\mu=2,\sigma=1)])

There are 3 different ways of declaring the SCM:

## 1. List Of Strings
Describe the assignments as strings of the form:

'VAR = FUNC(Noise, parent1, parent2, ...), Noise ~ DistributionXYZ'

Note that - out of convenience - in this case, one does not need to (and isn't allowed to)
restate the noise symbol string in the distribution (as would otherwise be necessary
in constructing sympy distributions).


```python
from scmodels import SCM

myscm = SCM(
    [
        "Z = N, N ~ LogLogistic(alpha=1, beta=1)",
        "X = N * 3 * Z ** 2, N ~ LogNormal(mean=1, std=1)",
        "Y = N + 2 * Z + sqrt(X), N ~ Normal(mean=2, std=1)"
    ]
)
```

## 2. Assignment Map

One can construct the SCM via an assignment map with the variables as keys and a tuple of the form:
(assignment string (as in 1.), noise distribution)

Note that the name of the noise symbol in the sympy.symbol constructor
(e.g. `Normal("N", mean=2, std=1)`) has to align with the noise variable
name (`N`) inside the assignment string.


```python
from sympy.stats import LogLogistic, LogNormal, Normal


assignment_map = {
   "Z": (
       "N",
       LogLogistic("N", alpha=1, beta=1)
   ),
   "X": (
       "N * 3 * Z ** 2",
       LogNormal("N", mean=1, std=1),
   ),
   "Y": (
       "N + 2 * Z + sqrt(X)",
       Normal("N", mean=2, std=1),
   ),
}

myscm = SCM(assignment_map)
```

## 3. Functional Map

One can also declare the SCM via specifying the variable assignment in a dictionary with the
variables as keys and as values a sequence of length 3 of the form:

(parent strings list, Callable, Noise distribution)

This allows the user to supply complex functions outside the space of predefined functions.

The SCM supports a form of pretty printing its current setup, which includes mentioning active interventions
and the assignments

# Features

## Prettyprint

The SCM supports a form of informative printing of its current setup,
which includes mentioning active interventions and the assignments.
The function declaration won't be informative if the SCM has been
constructed with custom Callables.


```python
print(myscm)
```

    Structural Causal Model of 3 variables: Z, X, Y
    Following variables are actively intervened on: []
    Current Assignments are:
    Z := f(N) = N	 [ N ~ LogLogistic(alpha=1, beta=1) ]
    X := f(N, Z) = N * 3 * Z ** 2	 [ N ~ LogNormal(mean=1, std=1) ]
    Y := f(N, X, Z) = N + 2 * Z + sqrt(X)	 [ N ~ Normal(mean=2, std=1) ]


## Interventions
One can easily perform interventions on the variables,
e.g. a Do-intervention or also general interventions, which remodel the connections, assignments, and noise distributions.
For the general case, the sample construction possibilities apply as for the SCM constructor.

For the example of the do-intervention ![\text{do}(X=1=)](https://latex.codecogs.com/svg.latex?&space;\text{do}(X=1)), simply execute


```python
myscm.do_intervention([("X", 1)])
```

and restore the original state with


```python
myscm.und_interventions()
```

## Sampling

Once can sample as many samples from the SCM as needed through the method `myscm.sample(n)`.


```python
n = 5
myscm.sample(n)
```

If infinite sampling is desired, one can also receive a sampling generator through


```python
container = {var: [] for var in myscm}
sampler = myscm.sample_iter(container)
```

`container` is an optional target dictionary to store the computed samples in.


```python
for i in range(n):
    next(sampler)

container
```

If the target container is not provided, the generator returns a new `dict` for every sample.


```python
next(myscm.sample_iter())
```

    C:\Users\Michael\.conda\envs\py38\lib\site-packages\sympy\stats\rv.py:1104: UserWarning: 
    The return type of sample has been changed to return an iterator
    object since version 1.7. For more information see
    https://github.com/sympy/sympy/issues/19061
      warnings.warn(filldedent(message))





<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>X</th>
      <th>Z</th>
      <th>Y</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>0.037574</td>
      <td>2.891794</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>0.145460</td>
      <td>0.049132</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1</td>
      <td>0.831458</td>
      <td>3.944336</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1</td>
      <td>6.835975</td>
      <td>16.333121</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1</td>
      <td>3.091106</td>
      <td>9.645460</td>
    </tr>
  </tbody>
</table>
</div>



## Plotting
If you have graphviz installed, you can plot the DAG by calling

```python
myscm.plot(node_size=1000, alpha=1)
```

![example_plot](https://github.com/maichmueller/scm/blob/master/docs/images/example_plot.png)


```python

```
