

<div align="center">
<img src="https://raw.githubusercontent.com/google/jax/master/images/jax_logo_250px.png" alt="logo"></img>
</div>

# JAX: Autograd and XLA [![Test status](https://travis-ci.org/google/jax.svg?branch=master)](https://travis-ci.org/google/jax)

JAX is [Autograd](https://github.com/hips/autograd) and
[XLA](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/compiler/xla/g3doc/overview.md),
brought together for high-performance machine learning research.

With its updated version of [Autograd](https://github.com/hips/autograd),
JAX can automatically differentiate native
Python and NumPy functions. It can differentiate through loops, branches,
recursion, and closures, and it can take derivatives of derivatives of
derivatives. It supports reverse-mode differentiation (a.k.a. backpropagation)
via [`grad`](#automatic-differentiation-with-grad) as well as forward-mode differentiation,
and the two can be composed arbitrarily to any order.

What   new is that JAX uses
[XLA](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/compiler/xla/g3doc/overview.md)
to compile and run your NumPy programs on GPUs and TPUs. Compilation happens
under the hood by default, with library calls getting just-in-time compiled and
executed. But JAX also lets you just-in-time compile your own Python functions
into XLA-optimized kernels using a one-function API,
[`jit`](#compilation-with-jit). Compilation and automatic differentiation can be
composed arbitrarily, so you can express sophisticated algorithms and get
maximal performance without leaving Python.

Dig a little deeper, and you'll see that JAX is really an extensible system for
[composable function transformations](#transformations). Both
[`grad`](#automatic-differentiation-with-grad) and [`jit`](#compilation-with-jit)
are instances of such transformations. Another is [`vmap`](#auto-vectorization-with-vmap)
for automatic vectorization, with more to come.

This is a research project, not an official Google product. Expect bugs and
sharp edges. Please help by trying it out, [reporting
bugs](https://github.com/google/jax/issues), and letting us know what you
think!

```python
import jax.numpy as np
from jax import grad, jit, vmap
from functools import partial

def predict(params, inputs):
  for W, b in params:
    outputs = np.dot(inputs, W) + b
    inputs = np.tanh(outputs)
  return outputs

def logprob_fun(params, inputs, targets):
  preds = predict(params, inputs)
  return np.sum((preds - targets)**2)

grad_fun = jit(grad(logprob_fun))  # compiled gradient evaluation function
perex_grads = jit(vmap(grad_fun, in_axes=(None, 0, 0)))  # fast per-example grads
```

JAX started as a research project by [Matt Johnson](https://github.com/mattjj),
[Roy Frostig](https://github.com/froystig), [Dougal
Maclaurin](https://github.com/dougalm), and [Chris
Leary](https://github.com/learyg), and is now developed [in the
open](https://github.com/google/jax) by a growing number of
[contributors](#contributors).

### Contents
* [Quickstart: Colab in the Cloud](#quickstart-colab-in-the-cloud)
* [Installation](#installation)
* [Running the tests](#running-the-tests)
* [Reference documentation](#reference-documentation)
* [A brief tour](#a-brief-tour)
* [What   supported](#whats-supported)
* [Transformations](#transformations)
* [Random numbers are different](#random-numbers-are-different)
* [Mini-libraries](#mini-libraries)
* [How it works](#how-it-works)
* [What we're working on](#what-were-working-on)
* [Current gotchas](#current-gotchas)

## Quickstart: Colab in the Cloud
Jump right in using a notebook in your browser, connected to a Google Cloud GPU:
- [The basics: NumPy on accelerators, `grad` for differentiation, `jit` for compilation, and `vmap` for vectorization](https://colab.research.google.com/github/google/jax/blob/master/notebooks/quickstart.ipynb)
- [Training a Simple Neural Network, with PyTorch Data Loading](https://colab.research.google.com/github/google/jax/blob/master/notebooks/neural_network_and_data_loading.ipynb)

## Installation
JAX is written in pure Python, but it depends on XLA, which needs to be
compiled and installed as the `jaxlib` package. Use the following instructions
to build JAX from source or install a binary package with pip.

### Building JAX from source
First, obtain the JAX source code, and make sure `scipy` is installed.

```bash
git clone https://github.com/google/jax
cd jax
pip install scipy
```

If you are building on a Mac, make sure XCode and the XCode command line tools
are installed.

To build XLA with CUDA support, you can run

```bash
python build/build.py --enable_cuda
pip install -e build  # install jaxlib (includes XLA)
pip install -e .      # install jax (pure Python)
```

See `python build/build.py --help` for configuration options, including ways to
specify the paths to CUDA and CUDNN, which you must have installed. The build
also depends on NumPy, and a compiler toolchain corresponding to that of
Ubuntu 16.04 or newer.

To build XLA without CUDA GPU support (CPU only), drop the `--enable_cuda`:

```bash
python build/build.py
pip install -e build  # install jaxlib (includes XLA)
pip install -e .      # install jax
```

To upgrade to the latest version from GitHub, just run `git pull` from the JAX
repository root, and rebuild by running `build.py` if necessary. You shouldn't have
to reinstall because `pip install -e` sets up symbolic links from site-packages
into the repository.

### pip installation

Installing XLA with prebuilt binaries via `pip` is still experimental,
especially with GPU support. Let us know on [the issue
tracker](https://github.com/google/jax/issues) if you run into any errors.

To install a CPU-only version, which might be useful for doing local
development on a laptop, you can run

```bash
pip install --upgrade jax jaxlib  # CPU-only version
```

If you want to install JAX with both CPU and GPU support, using existing CUDA
and CUDNN7 installations on your machine (for example, preinstalled on your
cloud VM), you can run

```bash
# install jaxlib
PYTHON_VERSION=cp27  # alternatives: cp27, cp36, cp37
CUDA_VERSION=cuda92  # alternatives: cuda90, cuda92, cuda100
PLATFORM=linux_x86_64  # alternatives: linux_x86_64
BASE_URL='https://storage.googleapis.com/jax-wheels'
pip install --upgrade $BASE_URL/$CUDA_VERSION/jaxlib-0.1.7-$PYTHON_VERSION-none-$PLATFORM.whl

pip install --upgrade jax  # install jax
```

The library package name must correspond to the version of the existing CUDA
installation you want to use, with `cuda100` for CUDA 10.0, `cuda92` for CUDA
9.2, and `cuda90` for CUDA 9.0. To find your CUDA and CUDNN versions, you can
run commands like these, depending on your CUDNN install path:

```bash
nvcc --version
grep CUDNN_MAJOR -A 2 /usr/local/cuda/include/cudnn.h  # might need different path
```

The Python version must match your Python interpreter. There are prebuilt wheels
for Python 2.7, 3.6, and 3.7; for anything else, you must build from source.


## Running the tests

To run all the JAX tests, from the repository root directory run

```bash
nosetests tests
```

JAX generates test cases combinatorially, and you can control the number of
cases that are generated and checked for each test (default 10):

```bash
JAX_NUM_GENERATED_CASES=100 nosetests tests
```

You can run a more specific set of tests using
[`nose`](https://nose.readthedocs.io/en/latest/usage.html)   built-in selection
mechanisms, or alternatively you can run a specific test file directly to see
more detailed information about the cases being run:

```bash
python tests/lax_numpy_test.py --num_generated_cases=5
```

## Reference documentation

For details about the JAX API, see the
[reference documentation](https://jax.readthedocs.io/).

## A brief tour

```python
In [1]: import jax.numpy as np

In [2]: from jax import random

In [3]: key = random.PRNGKey(0)

In [4]: x = random.normal(key, (5000, 5000))

In [5]: print(np.dot(x, x.T) / 2)  # fast!
[[  2.52727051e+03   8.15895557e+00  -8.53276134e-01 ...,  # ...

In [6]: print(np.dot(x, x.T) / 2)  # even faster!
[[  2.52727051e+03   8.15895557e+00  -8.53276134e-01 ...,  # ...
```

happening e complete toy
example:

```python
from jax import grad, jit
import jax.numpy as np

def sigmoid(x):
    return 0.5 * (np.tanh(x / 2.) + 1)

# Outputs probability of a label being true according to logistic model.
def logistic_predictions(weights, inputs):
    return sigmoid(np.dot(inputs, weights))

# Training loss is the negative log-likelihood of the training labels.
def loss(weights, inputs, targets):
    preds = logistic_predictions(weights, inputs)
    label_logprobs = np.log(preds) * targets + np.log(1 - preds) * (1 - targets)
    return -np.sum(label_logprobs)

# Build a toy dataset.
inputs = np.array([[0.52, 1.12,  0.77],
                   [0.88, -1.08, 0.15],
                   [0.52, 0.06, -1.30],
                   [0.74, -2.49, 1.39]])
targets = np.array([True, True, False, True])

# Define a compiled function that returns gradients of the training loss
training_gradient_fun = jit(grad(loss))

# Optimize weights using gradient descent.
weights = np.array([0.0, 0.0, 0.0])
print("Initial loss: {:0.2f}".format(loss(weights, inputs, targets)))
for i in range(100):
    weights -= 0.1 * training_gradient_fun(weights, inputs, targets)

print("Trained loss: {:0.2f}".format(loss(weights, inputs, targets)))
```

To see more, check out the [quickstart
notebook](https://colab.research.google.com/github/google/jax/blob/master/notebooks/quickstart.ipynb),
a [simple MNIST classifier
example](https://github.com/google/jax/blob/master/examples/mnist_classifier.py)
and the rest of the [JAX
examples](https://github.com/google/jax/blob/master/examples/).

## What   supported

I just as an accelerator-backed NumPy, without using `grad` or
`jit` in your code, then in principle there are no constraints, though some
NumPy ented yet. A list of supported functions can
be found in the [reference documentation](https://jax.readthedocs.io/).

Generally using `np.dot(A, B)` is
better than `A.dot(B)` because the former gives us more opportunities to run the
computation on the device. NumPy also does a lot of work to cast any array-like
function arguments to arrays, as in `np.sum([x, y])`, while `jax.numpy`
typically requires explicit casting of array arguments, like
`np.sum(np.array([x, y]))`.

For automatic differentiation with `grad`, JAX has the same restrictions
as [Autograd](https://github.com/hips/autograd). Specifically, differentiation
works with indexing (`x = A[i, j, :]`) but not indexed assignment (`A[i, j] =
x`) or indexed in-place updating (`A[i] += b`). You can use lists, tuples, and
dicts freely: JAX doesn't even see them. Using `np.dot(A, B)` rather than
`A.dot(B)` is required for automatic differentiation when `A` is a raw ndarray.



In general, JAX is intended to be used with a functional style of Python
programming. Functions passed to transformations like `grad` and `jit` are
expected to be free of side-effects. You can write print statements for
debugging but they may only be executed once if they're under a `jit` decorator.

> TLDR **Do use**
>
> *   Functional programming
>     functions (help us add more!)
> *   [Some](https://jax.readthedocs.io/en/latest/jax.scipy.html) SciPy functions
> *   Indexing and slicing of arrays like `x = A[[5, 1, 7], :, 2:4]`
> *   Explicit array creation from lists like `A = np.array([x, y])`
>
> **Do not use**
>
> *   Assignment into arrays like `A[0, 0] = x`
> *   Implicit casting to arrays like `np.sum([x, y])` (use `np.sum(np.array([x,
>     y])` instead)
> *   `A.dot(B)` method syntax for functions of more than one argument (use
>     `np.dot(A, B)` instead)
> *   Side-effects like mutation of arguments or mutation of global variables
> *   The `out` argument of NumPy functions
> *   Dtype casting like `np.float64(x)` (use `x.astype('float64')` or
>     `x.astype(np.float64)` instead).
>
> **For jit functions, do not**
>
> *   Control flow based on dynamic values `if x > 0: ...`. Control flow based
>     on shapes is fine: `if x.shape[0] > 2: ...` and `for subarr in array`.
> *   Slicing `A[i:i+5]` for dynamic index `i` (use `lax.dynamic_slice` instead)
>     or boolean indexing `A[bool_ind]` for traced values `bool_ind`.

You should get loud errors if your code violates any of these.

## Transformations

At its core, JAX is an extensible system for transforming numerical functions.
We currently expose three important transformations: `grad`, `jit`, and `vmap`.

### Automatic differentiation with grad

JAX has roughly the same API as [Autograd](https://github.com/hips/autograd).
The most popular function is `grad` for reverse-mode gradients:

```python
from jax import grad
import jax.numpy as np

def tanh(x):  # Define a function
  y = np.exp(-2.0 * x)
  return (1.0 - y) / (1.0 + y)

grad_tanh = grad(tanh)  # Obtain its gradient function
print(grad_tanh(1.0))   # Evaluate it at x = 1.0
# prints 0.41997434161402603
```

You can differentiate to any order with `grad`.

For more advanced autodiff, you can use `jax.vjp` for reverse-mode
vector-Jacobian products and `jax.jvp` for forward-mode Jacobian-vector
products. The two can be composed arbitrarily with one another, and with other
JAX transformations. Here   one way to compose
those to make a function that efficiently computes full Hessian matrices:

```python
from jax import jit, jacfwd, jacrev
def hessian(fun):
  return jit(jacfwd(jacrev(fun)))
```

As with Autograd, you're free to use differentiation with Python control
structures:

```python
def abs_val(x):
  if x > 0:
    return x
  else:
    return -x

abs_val_grad = grad(abs_val)
print(abs_val_grad(1.0))   # prints 1.0
print(abs_val_grad(-1.0))  # prints -1.0 (abs_val is re-evaluated)
```

### Compilation with jit

You can use XLA to compile your functions end-to-end with `jit`, used either as
an `@jit` decorator or as a higher-order function.

```python
import jax.numpy as np
from jax import jit

def slow_f(x):
  # Element-wise ops see a large benefit from fusion
  return x * x + x * 2.0

x = np.ones((5000, 5000))
fast_f = jit(slow_f)
%timeit -n10 -r3 fast_f(x)  # ~ 4.5 ms / loop on Titan X
%timeit -n10 -r3 slow_f(x)  # ~ 14.5 ms / loop (also on GPU via JAX)
```

You can mix `jit` and `grad` and any other JAX transformation however you like.

### Auto-vectorization with vmap

`vmap` is the vectorizing map.
It has the familiar semantics of mapping a function along array axes, but
instead of keeping the loop on the out

Using `vmap` can save you from having to carry around batch dimensions in your
code. For example, consider this simple *unbatched* neural network prediction
function:

```python
def predict(params, input_vec):
  assert input_vec.ndim == 1
  for W, b in params:
    output_vec = np.dot(W, input_vec) + b  # `input_vec` on the right-hand side!
    input_vec = np.tanh(output_vec)
  return output_vec
```


```python
from functools import partial
predictions = np.stack(list(map(partial(predict, params), input_batch)))
```


The `vmap` function does that transformation for us. That is, if we write

```python
from jax import vmap
predictions = vmap(partial(predict, params))(input_batch)
# or, alternatively
predictions = vmap(predict, in_axes=(None, 0))(params, input_batch)
```


```python
per_example_gradients = vmap(partial(grad(loss), params))(inputs, targets)
```

Of course, `vmap` can be arbitrarily composed with `jit`, `grad`, and any other
JAX transformation! We use `vmap` with both forward- and reverse-mode automatic
differentiation for fast Jacobian and Hessian matrix calculations in
`jax.jacfwd`, `jax.jacrev`, and `jax.hessian`.


## Random numbers are different


The `jax.random` library uses
[count-based PRNGs](http://www.thesalmons.org/john/random123/papers/random123sc11.pdf)
and a functional array-oriented
[splitting model](http://publications.lib.chalmers.se/records/fulltext/183348/local_183348.pdf).
To generate random values, you call a function like `jax.random.normal` and give
it a PRNG key:

```python
import jax.random as random

key = random.PRNGKey(0)
print(random.normal(key, shape=(3,)))  # [ 1.81608593 -0.48262325  0.33988902]
```

If we make the same call again with the same key, we get the same values:

```python
print(random.normal(key, shape=(3,)))  # [ 1.81608593 -0.48262325  0.33988902]
```

The key never gets updated. So how do we get fresh random values? We use
`jax.random.split` to create new keys from existing ones. A common pattern is to
split off a new key for every function call that needs random values:

```python
key = random.PRNGKey(0)

key, subkey = random.split(key)
print(random.normal(subkey, shape=(3,)))  # [ 1.1378783  -1.22095478 -0.59153646]

key, subkey = random.split(key)
print(random.normal(subkey, shape=(3,)))  # [-0.06607265  0.16676566  1.17800343]
```

By splitting the PRNG key, not only do we avoid having to thread random states
back out of every function call, but also we can generate multiple random arrays
in parallel because we can avoid unnecessary sequential dependencies.

Therea gotcha here, which is that it easy to unintentionally reuse a key
without splitting. We intend to add a check for this (a sort of dynamic linear
typing) but for now it something to be careful about.

For more detailed information on the design and the reasoning behind it, see the
[PRNG design doc](design_notes/prng.md).


## Mini-libraries

JAX provides some small, experimental libraries for machine learning. These
libraries are in part about providing tools and in part about serving as
examples for how to build such libraries using JAX. Each one is only a few
hundred lines of code, so take a look inside and adapt them as you need!

### Neural-net building with Stax

**Stax** is a functional neural network building library. The basic idea is that
a single layer or an entire network can be modeled as an `(init_fun, apply_fun)`
pair. The `init_fun` is used to initialize network parameters and the
`apply_fun` takes parameters and inputs to produce outputs. There are
constructor functions for common basic pairs, like `Conv` and `Relu`, and these
pairs can be composed in series using `stax.serial` or in parallel using
`stax.parallel`.

 example:

```python
import jax.numpy as np
from jax.experimental import stax
from jax.experimental.stax import Conv, Dense, MaxPool, Relu, Flatten, LogSoftmax

# Use stax to set up network initialization and evaluation functions
net_init, net_apply = stax.serial(
    Conv(32, (3, 3), padding='SAME'), Relu,
    Conv(64, (3, 3), padding='SAME'), Relu,
    MaxPool((2, 2)), Flatten,
    Dense(128), Relu,
    Dense(10), LogSoftmax,
)

# Initialize parameters, not committing to a batch shape
in_shape = (-1, 28, 28, 1)
out_shape, net_params = net_init(in_shape)

# Apply network to dummy inputs
inputs = np.zeros((128, 28, 28, 1))
predictions = net_apply(net_params, inputs)
```

### First-order optimization

JAX has a minimal optimization library focused on stochastic first-order
optimizers. Every optimizer is modeled as an `(init_fun, update_fun)` pair. The
`init_fun` is used to initialize the optimizer state, which could include things
like momentum variables, and the `update_fun` accepts a gradient and an
optimizer state to produce a new optimizer state. The parameters being optimized
can be ndarrays or arbitrarily-nested list/tuple/dict structures, so you can
store your parameters howev.

Heple, using `jit` to compile the whole update end-to-end:

```python
from jax.experimental import optimizers
from jax import jit, grad

# Define a simple squared-error loss
def loss(params, batch):
  inputs, targets = batch
  predictions = net_apply(params, inputs)
  return np.sum((predictions - targets)**2)

# Use optimizers to set optimizer initialization and update functions
opt_init, opt_update = optimizers.momentum(step_size=1e-3, mass=0.9)

# Define a compiled update step
@jit
def step(i, opt_state, batch):
  params = optimizers.get_params(opt_state)
  g = grad(loss)(params, batch)
  return opt_update(i, g, opt_state)

# Dummy input data stream
data_generator = ((np.zeros((128, 28, 28, 1)), np.zeros((128, 10)))
                  for _ in range(10))

# Optimize parameters in a loop
opt_state = opt_init(net_params)
for i in range(10):
  opt_state = step(i, opt_state, next(data_generator))
net_params = optimizers.get_params(opt_state)
```

## How it works

Programming in machine learning is about expressing and transforming functions.
Transformations include automatic differentiation, compilation for accelerators,
and automatic batching. High-level languages like Python are great for
expressing functions, but usually all we can do with them is apply them. We lose
access to their internal structure which would let us perform transformations.

JAX is a tool for specializing and translating high-level Python+NumPy functions
into a representation that can be transformed and then lifted back into a Python
function.

![simplified-lifecycle](https://raw.githubusercontent.com/google/jax/master/images/lifecycle.png)

JAX specializes Python functions by tracing. Tracing a function means monitoring
all the basic  To keep track
of how data flows between these primitives, values being tracked are wrapped in
instances of the `Tracer` class.

When a Python function is provido go! 
See for more
information about `jit` requirements.

The good news about this tradeoff is that `jit` is opt-in: JAX libraries use
`jit` on individual operations and functions behind the scenes, allowing you to
write unrestricted Python+Numpy and still make use of a hardware accelerator.
But when you want to maximize performance, you can often use `jit` in your own
code to compile and end-to-end optimize much bigger functions.

## What we're working on
1. Documentation!
2. Cloud TPU support
3. Multi-GPU and multi-TPU support
4. Full NumPy coverage and some SciPy coverage
5. Full coverage for vmap
6. Make everything faster
    * Lowering the XLA function dispatch overhead
    * Linear algebra routines (MKL on CPU, MAGMA on GPU)
7. `cond` and `while` primitives with efficient automatic differentiation

## Current gotchas

Some things we don't handle that might surprise NumPy users:
1. No in-place mutation syntax. JAX requires functional code. You can use
   lax.dynamic\_update\_slice for slice updating that, under `@jit`, will be
   optimized to in-place buffer updating.
2. PRNGs can be awkward, and linearity is not checked with a warning.

## Contributors

So far, JAX includes lots of help and contributions from
[Jamie Townsend](https://github.com/j-towns),
[Peter Hawkins](https://github.com/hawkinsp),
[Jonathan Ragan-Kelley](https://people.eecs.berkeley.edu/~jrk/),
[Alex Wiltschko](http://github.com/alexbw),
George Dahl,
[Stephan Hoyer](http://stephanhoyer.com/),
Sam Schoenholz,
[Eli Bendersky](https://github.com/eliben),
Zak Stone,
[Alexey Radul](https://github.com/axch),
Michael Isard,
Skye Wanderman-Milne,
and many others.

<br>
<br>
##Submit Solution
---
<div class="jumbotron jumbotron-fluid bg-light text-dark">
  <div class="container">
    <form>
        <div class="row">
            <div class="col">
            <input type="text" class="form-control" placeholder="Name">
            </div>
        </div>
        <br>
        <div class="form-group">
            <input type="email" class="form-control" id="exampleFormControlInput1" placeholder="name@example.com">
        </div>
        <div class="form-group">
            <label for="exampleFormControlTextarea1">Comments (Optional):</label>
            <textarea class="form-control" id="exampleFormControlTextarea1" rows="3"></textarea>
        </div>
        <div class="form-group">
            <input type="file" class="form-control-file" id="exampleFormControlFile1">
        </div>
        <button type="submit" class="btn btn-primary">Submit</button>
    </form>
  </div>
</div>