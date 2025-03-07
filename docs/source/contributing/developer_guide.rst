:orphan:

..
    _referenced in html_theme_options docs/source/conf.py

====================
PyMC Developer Guide
====================

:doc:`PyMC <index>` is a Python package for Bayesian
statistical modeling built on top of
:doc:`Aesara <aesara:index>`. This
document aims to explain the design and implementation of probabilistic
programming in PyMC, with comparisons to other PPL like TensorFlow Probability (TFP)
and Pyro in mind. A user-facing API
introduction can be found in the :ref:`API quickstart <pymc_overview>`. A more accessible, user facing deep introduction can be found in
`Peadar Coyle's probabilistic programming primer <https://github.com/springcoil/probabilisticprogrammingprimer>`__

Distribution
------------

A high-level introduction of ``Distribution`` in PyMC can be found in
the :ref:`documentation <api_distributions>`. The source
code of the probability distributions is nested under
:ref:`pymc/distributions <api_distributions>`,
with the :class:`~pymc.Distribution` class defined in ``distribution.py``.
A few important points to highlight in the Distribution Class:

.. code:: python

    class Distribution:
        """Statistical distribution"""
        def __new__(cls, name, *args, **kwargs):
            ...
            try:
                model = Model.get_context()
            except TypeError:
                raise TypeError(...

            if isinstance(name, string_types):
                ...
                dist = cls.dist(*args, **kwargs)
                return model.Var(name, dist, ...)
            ...

In a way, the snippet above represents the unique features of pymc's
``Distribution`` class:

- Distribution objects are only usable inside of a ``Model`` context. If they are created outside of the model context manager, it raises an error.

- A ``Distribution`` requires at least a name argument, and other parameters that defines the Distribution.

- When a ``Distribution`` is initialized inside of a Model context, two things happen:

  1. a stateless distribution is initialized ``dist = {DISTRIBUTION_cls}.dist(*args, **kwargs)``;
  2. a random variable following the said distribution is added to the model ``model.Var(name, dist, ...)``

Thus, users who are building models using ``with pm.Model() ...`` should
be aware that they are never directly exposed to static and stateless
distributions, but rather random variables that follow some density
functions. Instead, to access a stateless distribution, you need to call
``pm.SomeDistribution.dist(...)`` or ``RV.dist`` *after* you initialized
``RV`` in a model context (see
https://docs.pymc.io/Probability\_Distributions.html#using-pymc-distributions-without-a-model).

With this distinction in mind, we can take a closer look at the
stateless distribution part of pymc (see distribution api in :ref:`doc <api_distributions>`), which divided into:

- Continuous

- Discrete

- Multivariate

- Mixture

- Timeseries

Quote from the doc:

    All distributions in ``pm.distributions`` will have two important
    methods: ``random()`` and ``logp()`` with the following signatures:

.. code:: python

    class SomeDistribution(Continuous):
        def __init__(...):
            ...

        def random(self, point=None, size=None):
            ...
            return random_samples

        def logp(self, value):
            ...
            return total_log_prob

PyMC expects the ``logp()`` method to return a log-probability
evaluated at the passed value argument. This method is used internally
by all of the inference methods to calculate the model log-probability,
which is then used for fitting models. The ``random()`` method is
used to simulate values from the variable, and is used internally for
posterior predictive checks.

In the PyMC ``Distribution`` class, the ``logp()`` method is the most
elementary. As long as you have a well-behaved density function, we can
use it in the model to build the model log-likelihood function. Random
number generation is great to have, but sometimes there might not be
efficient random number generator for some densities. Since a function
is all you need, you can wrap almost any Aesara function into a
distribution using ``pm.DensityDist``
https://docs.pymc.io/Probability\_Distributions.html#custom-distributions

Thus, distributions that are defined in the ``distributions`` submodule
(e.g. look at ``pm.Normal`` in ``pymc.distributions.continuous``), each
describes a *family* of probabilistic distribution (no different from
distribution in other PPL library). Once it is initialised within a
model context, it contains properties that are related to the random
variable (*e.g.* mean/expectation). Note that if the parameters are
constants, these properties could be the same as the distribution
properties.

Reflection
~~~~~~~~~~

How tensor/value semantics for probability distributions is enabled in pymc:

In PyMC, we treat ``x = Normal('x', 0, 1)`` as defining a random
variable (intercepted and collected under a model context, more on that
below), and x.dist() as the associated density/mass function
(distribution in the mathematical sense). It is not perfect, and now
after a few years learning Bayesian statistics I also realized these
subtleties (i.e., the distinction between *random variable* and
*distribution*).

But when I was learning probabilistic modelling as a
beginner, I did find this approach to be the easiest and most
straightforward. In a perfect world, we should have
:math:`x \sim \text{Normal}(0, 1)` which defines a random variable that
follows a Gaussian distribution, and
:math:`\chi = \text{Normal}(0, 1), x \sim \chi` which define a `probability
density function <https://en.wikipedia.org/wiki/Probability_density_function>`__ that takes input :math:`x`

.. math::
    X:=f(x) = \frac{1}{\sigma \sqrt{2 \pi}} \exp^{- 0.5 (\frac{x - \mu}{\sigma})^2}\vert_{\mu = 0, \sigma=1} = \frac{1}{\sqrt{2 \pi}} \exp^{- 0.5 x^2}

Within a model context, RVs are essentially Aesara tensors (more on that
below). This is different than TFP and pyro, where you need to be more
explicit about the conversion. For example:

**PyMC**

.. code:: python

    with pm.Model() as model:
        z = pm.Normal('z', mu=0., sigma=5.)             # ==> aesara.tensor.var.TensorVariable
        x = pm.Normal('x', mu=z, sigma=1., observed=5.) # ==> aesara.tensor.var.TensorVariable
    x.logp({'z': 2.5})                                  # ==> -4.0439386
    model.logp({'z': 2.5})                              # ==> -6.6973152

**TFP**

.. code:: python

    import tensorflow.compat.v1 as tf
    from tensorflow_probability import distributions as tfd

    with tf.Session() as sess:
        z_dist = tfd.Normal(loc=0., scale=5.)            # ==> <class 'tfp.python.distributions.normal.Normal'>
        z = z_dist.sample()                              # ==> <class 'tensorflow.python.framework.ops.Tensor'>
        x = tfd.Normal(loc=z, scale=1.).log_prob(5.)     # ==> <class 'tensorflow.python.framework.ops.Tensor'>
        model_logp = z_dist.log_prob(z) + x
        print(sess.run(x, feed_dict={z: 2.5}))           # ==> -4.0439386
        print(sess.run(model_logp, feed_dict={z: 2.5}))  # ==> -6.6973152

**pyro**

.. code:: python

    z_dist = dist.Normal(loc=0., scale=5.)           # ==> <class 'pyro.distributions.torch.Normal'>
    z = pyro.sample("z", z_dist)                     # ==> <class 'torch.Tensor'>
    # reset/specify value of z
    z.data = torch.tensor(2.5)
    x = dist.Normal(loc=z, scale=1.).log_prob(5.)    # ==> <class 'torch.Tensor'>
    model_logp = z_dist.log_prob(z) + x
    x                                                # ==> -4.0439386
    model_logp                                       # ==> -6.6973152


``logp`` method, very different behind the curtain
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``logp`` method is straightforward - it is an Aesara function within each
distribution. It has the following signature:

.. code:: python

    def logp(self, value):
        # GET PARAMETERS
        param1, param2, ... = self.params1, self.params2, ...
        # EVALUATE LOG-LIKELIHOOD FUNCTION, all inputs are (or array that could be convert to) Aesara tensor
        total_log_prob = f(param1, param2, ..., value)
        return total_log_prob

In the ``logp`` method, parameters and values are either Aesara tensors,
or could be converted to tensors. It is rather convenient as the
evaluation of logp is represented as a tensor (``RV.logpt``), and when
we linked different ``logp`` together (e.g., summing all ``RVs.logpt``
to get the model totall logp) the dependence is taken care of by Aesara
when the graph is built and compiled. Again, since the compiled function
depends on the nodes that already in the graph, whenever you want to generate
a new function that takes new input tensors you either need to regenerate the graph
with the appropriate dependencies, or replace the node by editing the existing graph.
In PyMC we use the second approach by using ``aesara.clone_replace()`` when it is needed.

As explained above, distribution in a ``pm.Model()`` context
automatically turn into a tensor with distribution property (pymc
random variable). To get the logp of a free\_RV is just evaluating the
``logp()`` `on
itself <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/model.py#L1212-L1213>`__:

.. code:: python

        # self is a aesara.tensor with a distribution attached
        self.logp_sum_unscaledt = distribution.logp_sum(self)
        self.logp_nojac_unscaledt = distribution.logp_nojac(self)

Or for an observed RV. it evaluate the logp on the data:

.. code:: python

        self.logp_sum_unscaledt = distribution.logp_sum(data)
        self.logp_nojac_unscaledt = distribution.logp_nojac(data)

Model context and Random Variable
---------------------------------

I like to think that the ``with pm.Model() ...`` is a key syntax feature
and *the* signature of PyMC model language, and in general a great
out-of-the-box thinking/usage of the context manager in Python (with
`some
critics <https://twitter.com/_szhang/status/890793373740617729>`__, of
course).

Essentially `what a context manager
does <https://www.python.org/dev/peps/pep-0343/>`__ is:

.. code:: python

    with EXPR as VAR:
        USERCODE

which roughly translates into this:

.. code:: python

    VAR = EXPR
    VAR.__enter__()
    try:
        USERCODE
    finally:
        VAR.__exit__()

or conceptually:

.. code:: python

    with EXPR as VAR:
        # DO SOMETHING
        USERCODE
        # DO SOME ADDITIONAL THINGS

So what happened within the ``with pm.Model() as model: ...`` block,
besides the initial set up ``model = pm.Model()``? Starting from the
most elementary:

Random Variable
~~~~~~~~~~~~~~~

From the above session, we know that when we call eg
``pm.Normal('x', ...)`` within a Model context, it returns a random
variable. Thus, we have two equivalent ways of adding random variable to
a model:


.. code:: python

    with pm.Model() as m:
        x = pm.Normal('x', mu=0., sigma=1.)


.. parsed-literal::

    print(type(x))                              # ==> <class 'aesara.tensor.var.TensorVariable'>
    print(m.free_RVs)                           # ==> [x]
    print(logpt(x, 5.0))                        # ==> Elemwise{switch,no_inplace}.0
    print(logpt(x, 5.).eval({}))                # ==> -13.418938533204672
    print(m.logp({'x': 5.}))                    # ==> -13.418938533204672


In general, if a variable has observations (``observed`` parameter), the RV is
an observed RV, otherwise if it has a ``transformed`` (``transform`` parameter)
attribute, it is a transformed RV otherwise, it will be the most elementary
form: a free RV.  Note that this means that random variables with observations
cannot be transformed.

..
   Below, I will take a deeper look into transformed RV. A normal user
   might not necessarily come in contact with the concept, since a
   transformed RV and ``TransformedDistribution`` are intentionally not
   user facing.

   Because in PyMC there is no bijector class like in TFP or pyro, we only
   have a partial implementation called ``Transform``, which implements
   Jacobian correction for forward mapping only (there is no Jacobian
   correction for inverse mapping). The use cases we considered are limited
   to the set of distributions that are bounded, and the transformation
   maps the bounded set to the real line - see
   :ref:`API quickstart <pymc_overview#Automatic-transforms-of-bounded-RVs>`.
   However, other transformations are possible.
   In general, PyMC does not provide explicit functionality to transform
   one distribution to another. Instead, a dedicated distribution is
   usually created in order to optimise performance. But getting a
   ``TransformedDistribution`` is also possible (see also in
   :ref:`doc <pymc_overview##Transformed-distributions-and-changes-of-variables>`):

   .. code:: python


       lognorm = Exp().apply(pm.Normal.dist(0., 1.))
       lognorm


   .. parsed-literal::

       <pymc.distributions.transforms.TransformedDistribution at 0x7f1536749b00>



   Now, back to ``model.RV(...)`` - things returned from ``model.RV(...)``
   are Aesara tensor variables, and it is clear from looking at
   ``TransformedRV``:

   .. code:: python

       class TransformedRV(TensorVariable):
           ...

   as for ``FreeRV`` and ``ObservedRV``, they are ``TensorVariable``\s with
   ``Factor`` as mixin:

   .. code:: python

       class FreeRV(Factor, TensorVariable):
           ...

   ``Factor`` basically `enable and assign the
   logp <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/model.py#L195-L276>`__
   (representated as a tensor also) property to an Aesara tensor (thus
   making it a random variable). For a ``TransformedRV``, it transforms the
   distribution into a ``TransformedDistribution``, and then ``model.Var`` is
   called again to added the RV associated with the
   ``TransformedDistribution`` as a ``FreeRV``:

   .. code:: python

           ...
           self.transformed = model.Var(
                       transformed_name, transform.apply(distribution), total_size=total_size)

   note: after ``transform.apply(distribution)`` its ``.transform``
   porperty is set to ``None``, thus making sure that the above call will
   only add one ``FreeRV``. In another word, you *cannot* do chain
   transformation by nested applying multiple transforms to a Distribution
   (however, you can use `Chain
   transformation <https://docs.pymc.io/notebooks/api_quickstart.html?highlight=chain%20transformation>`__).

   .. code:: python

       z = pm.LogNormal.dist(mu=0., sigma=1., transform=tr.Log)
       z.transform           # ==> pymc.distributions.transforms.Log


   .. code:: python

       z2 = Exp().apply(z)
       z2.transform is None  # ==> True



Additional things that ``pm.Model`` does
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In a way, ``pm.Model`` is a tape machine that records what is being
added to the model, it keeps track the random variables (observed or
unobserved) and potential term (additional tensor that to be added to
the model logp), and also deterministic transformation (as bookkeeping):
named\_vars, free\_RVs, observed\_RVs, deterministics, potentials,
missing\_values. The model context then computes some simple model
properties, builds a bijection mapping that transforms between
dictionary and numpy/Aesara ndarray, thus allowing the ``logp``/``dlogp`` functions
to have two equivalent versions: one takes a ``dict`` as input and the other
takes an ``ndarray`` as input. More importantly, a ``pm.Model()`` contains methods
to compile Aesara functions that take Random Variables (that are also
initialised within the same model) as input, for example:

.. code:: python

    with pm.Model() as m:
        z = pm.Normal('z', 0., 10., shape=10)
        x = pm.Normal('x', z, 1., shape=10)

    print(m.initial_point)
    print(m.dict_to_array(m.initial_point))  # ==> m.bijection.map(m.initial_point)
    print(m.bijection.rmap(np.arange(20)))


.. parsed-literal::

    {'z': array([0., 0., 0., 0., 0., 0., 0., 0., 0., 0.]), 'x': array([0., 0., 0., 0., 0., 0., 0., 0., 0., 0.])}
    [0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 0.]
    {'z': array([10., 11., 12., 13., 14., 15., 16., 17., 18., 19.]), 'x': array([0., 1., 2., 3., 4., 5., 6., 7., 8., 9.])}


.. code:: python

    list(filter(lambda x: "logp" in x, dir(pm.Model)))


.. parsed-literal::

    ['d2logp',
     'd2logp_nojac',
     'datalogpt',
     'dlogp',
     'dlogp_array',
     'dlogp_nojac',
     'fastd2logp',
     'fastd2logp_nojac',
     'fastdlogp',
     'fastdlogp_nojac',
     'fastlogp',
     'fastlogp_nojac',
     'logp',
     'logp_array',
     'logp_dlogp_function',
     'logp_elemwise',
     'logp_nojac',
     'logp_nojact',
     'logpt',
     'varlogpt']



Logp and dlogp
--------------

The model collects all the random variables (everything in
``model.free_RVs`` and ``model.observed_RVs``) and potential term, and
sum them together to get the model logp:

.. code:: python

    @property
    def logpt(self):
        """Aesara scalar of log-probability of the model"""
        with self:
            factors = [var.logpt for var in self.basic_RVs] + self.potentials
            logp = at.sum([at.sum(factor) for factor in factors])
            ...
            return logp

which returns an Aesara tensor that its value depends on the free
parameters in the model (i.e., its parent nodes from the Aesara
graph).You can evaluate or compile into a python callable (that you can
pass numpy as input args). Note that the logp tensor depends on its
input in the Aesara graph, thus you cannot pass new tensor to generate a
logp function. For similar reason, in PyMC we do graph copying a lot
using aesara.clone_replace to replace the inputs to a tensor.

.. code:: python

    with pm.Model() as m:
        z = pm.Normal('z', 0., 10., shape=10)
        x = pm.Normal('x', z, 1., shape=10)
        y = pm.Normal('y', x.sum(), 1., observed=2.5)

    print(m.basic_RVs)    # ==> [z, x, y]
    print(m.free_RVs)     # ==> [z, x]


.. code:: python

    type(m.logpt)         # ==> aesara.tensor.var.TensorVariable


.. code:: python

    m.logpt.eval({x: np.random.randn(*x.tag.test_value.shape) for x in m.free_RVs})

output:

.. parsed-literal::

    array(-51.25369126)



PyMC then compiles a logp function with gradient that takes
``model.free_RVs`` as input and ``model.logpt`` as output. It could be a
subset of tensors in ``model.free_RVs`` if we want a conditional
logp/dlogp function:

.. code:: python

    def logp_dlogp_function(self, grad_vars=None, **kwargs):
        if grad_vars is None:
            grad_vars = list(typefilter(self.free_RVs, continuous_types))
        else:
            ...
        varnames = [var.name for var in grad_vars]  # In a simple case with only continous RVs,
                                                    # this is all the free_RVs
        extra_vars = [var for var in self.free_RVs if var.name not in varnames]
        return ValueGradFunction(self.logpt, grad_vars, extra_vars, **kwargs)

``ValueGradFunction`` is a callable class which isolates part of the
Aesara graph to compile additional Aesara functions. PyMC relies on
``aesara.clone_replace`` to copy the ``model.logpt`` and replace its input. It
does not edit or rewrite the graph directly.

The important parts of the above function is highlighted and commented.
On a high level, it allows us to build conditional logp function and its
gradient easily. Here is a taste of how it works in action:

.. code:: python

    inputlist = [np.random.randn(*x.tag.test_value.shape) for x in m.free_RVs]

    func = m.logp_dlogp_function()
    func.set_extra_values({})
    input_dict = {x.name: y for x, y in zip(m.free_RVs, inputlist)}
    print(input_dict)
    input_array = func.dict_to_array(input_dict)
    print(input_array)
    print(" ===== ")
    func(input_array)


.. parsed-literal::

    {'z': array([-0.7202002 ,  0.58712205, -1.44120196, -0.53153001, -0.36028732,
           -1.49098414, -0.80046792, -0.26351819,  1.91841949,  1.60004128]), 'x': array([ 0.01490006,  0.60958275, -0.06955203, -0.42430833, -1.43392303,
            1.13713493,  0.31650495, -0.62582879,  0.75642811,  0.50114527])}
    [-0.7202002   0.58712205 -1.44120196 -0.53153001 -0.36028732 -1.49098414
     -0.80046792 -0.26351819  1.91841949  1.60004128  0.01490006  0.60958275
     -0.06955203 -0.42430833 -1.43392303  1.13713493  0.31650495 -0.62582879
      0.75642811  0.50114527]
     =====
    (array(-51.0769075),
     array([ 0.74230226,  0.01658948,  1.38606194,  0.11253699, -1.07003284,
             2.64302891,  1.12497754, -0.35967542, -1.18117557, -1.11489642,
             0.98281586,  1.69545542,  0.34626619,  1.61069443,  2.79155183,
            -0.91020295,  0.60094326,  2.08022672,  2.8799075 ,  2.81681213]))



.. code:: python

    irv = 1
    print("Condition Logp: take %s as input and conditioned on the rest."%(m.free_RVs[irv].name))
    func_conditional = m.logp_dlogp_function(grad_vars=[m.free_RVs[irv]])
    func_conditional.set_extra_values(input_dict)
    input_array2 = func_conditional.dict_to_array(input_dict)
    print(input_array2)
    print(" ===== ")
    func_conditional(input_array2)


.. parsed-literal::

    Condition Logp: take x as input and conditioned on the rest.
    [ 0.01490006  0.60958275 -0.06955203 -0.42430833 -1.43392303  1.13713493
      0.31650495 -0.62582879  0.75642811  0.50114527]
     =====
    (array(-51.0769075),
     array([ 0.98281586,  1.69545542,  0.34626619,  1.61069443,  2.79155183,
            -0.91020295,  0.60094326,  2.08022672,  2.8799075 ,  2.81681213]))



So why is this necessary? One can imagine that we just compile one logp
function, and do bookkeeping ourselves. For example, we can build the
logp function in Aesara directly:

.. code:: python

    import aesara
    func = aesara.function(m.free_RVs, m.logpt)
    func(*inputlist)


.. parsed-literal::

    array(-51.0769075)



.. code:: python

    logpt_grad = aesara.grad(m.logpt, m.free_RVs)
    func_d = aesara.function(m.free_RVs, logpt_grad)
    func_d(*inputlist)


.. parsed-literal::

    [array([ 0.74230226,  0.01658948,  1.38606194,  0.11253699, -1.07003284,
             2.64302891,  1.12497754, -0.35967542, -1.18117557, -1.11489642]),
     array([ 0.98281586,  1.69545542,  0.34626619,  1.61069443,  2.79155183,
            -0.91020295,  0.60094326,  2.08022672,  2.8799075 ,  2.81681213])]



Similarly, build a conditional logp:

.. code:: python

    shared = aesara.shared(inputlist[1])
    func2 = aesara.function([m.free_RVs[0]], m.logpt, givens=[(m.free_RVs[1], shared)])
    print(func2(inputlist[0]))

    logpt_grad2 = aesara.grad(m.logpt, m.free_RVs[0])
    func_d2 = aesara.function([m.free_RVs[0]], logpt_grad2, givens=[(m.free_RVs[1], shared)])
    print(func_d2(inputlist[0]))


.. parsed-literal::

    -51.07690750130328
    [ 0.74230226  0.01658948  1.38606194  0.11253699 -1.07003284  2.64302891
      1.12497754 -0.35967542 -1.18117557 -1.11489642]


The above also gives the same logp and gradient as the output from
``model.logp_dlogp_function``. But the difficulty is to compile
everything into a single function:

.. code:: python

    func_logp_and_grad = aesara.function(m.free_RVs, [m.logpt, logpt_grad])  # ==> ERROR


We want to have a function that return the evaluation and its gradient
re each input: ``value, grad = f(x)``, but the naive implementation does
not work. We can of course wrap 2 functions - one for logp one for dlogp
- and output a list. But that would mean we need to call 2 functions. In
addition, when we write code using python logic to do bookkeeping when
we build our conditional logp. Using ``aesara.clone_replace``, we always have
the input to the Aesara function being a 1d vector (instead of a list of
RV that each can have very different shape), thus it is very easy to do
matrix operation like rotation etc.

Notes
~~~~~

| The current setup is quite powerful, as the Aesara compiled function
  is fairly fast to compile and to call. Also, when we are repeatedly
  calling a conditional logp function, external RV only need to reset
  once. However, there are still significant overheads when we are
  passing values between Aesara graph and numpy. That is the reason we
  often see no advantage in using GPU, because the data is copying
  between GPU and CPU at each function call - and for a small model, the
  result is a slower inference under GPU than CPU.
| Also, ``aesara.clone_replace`` is too convenient (pymc internal joke is that
  it is like a drug - very addictive). If all the operation happens in
  the graph (including the conditioning and setting value), I see no
  need to isolate part of the graph (via graph copying or graph
  rewriting) for building model and running inference.
| Moreover, if we are limiting to the problem that we can solved most
  confidently - model with all continous unknown parameters that could
  be sampled with dynamic HMC, there is even less need to think about
  graph cloning/rewriting.

Inference
---------

MCMC
~~~~

The ability for model instance to generate conditional logp and dlogp
function enable one of the unique feature of PyMC - `CompoundStep
method <https://docs.pymc.io/notebooks/sampling_compound_step.html>`__.
On a conceptual level it is a Metropolis-within-Gibbs sampler. User can
`specify different sampler of different
RVs <https://docs.pymc.io/notebooks/sampling_compound_step.html?highlight=compoundstep#Specify-compound-steps>`__.
Alternatively, it is implemented as yet another interceptor: the
``pm.sample(...)`` call will try to `assign the best step methods to
different
free\_RVs <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/sampling.py#L86-L152>`__
(e.g., NUTS if all free\_RVs are continous). Then, (conditional) logp
function(s) are compiled, and the sampler called each sampler within the
list of CompoundStep in a for-loop for one sample circle.

For each sampler, it implements a ``step.step`` method to perform MH
updates. Each time a dictionary (``point`` in ``PyMC`` land, same
structure as ``model.initial_point``) is passed as input and output a new
dictionary with the free\_RVs being sampled now has a new value (if
accepted, see
`here <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/step_methods/compound.py#L27>`__
and
`here <https://github.com/pymc-devs/pymc/blob/main/pymc/step_methods/compound.py>`__).
There are some example in the `CompoundStep
doc <https://docs.pymc.io/notebooks/sampling_compound_step.html#Specify-compound-steps>`__.

Transition kernel
^^^^^^^^^^^^^^^^^

The base class for most MCMC sampler (except SMC) is in
`ArrayStep <https://github.com/pymc-devs/pymc/blob/main/pymc/step_methods/arraystep.py>`__.
You can see that the ``step.step()`` is mapping the ``point`` into an
array, and call ``self.astep()``, which is an array in, array out
function. A pymc model compile a conditional logp/dlogp function that
replace the input RVs with a shared 1D tensor (flatten and stack view of
the original RVs). And the transition kernel (i.e., ``.astep()``) takes
array as input and output an array. See for example in the `MH
sampler <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/step_methods/metropolis.py#L139-L173>`__.

This is of course very different compare to the transition kernel in eg
TFP, which is a tenor in tensor out function. Moreover, transition
kernels in TFP do not flatten the tensors, see eg docstring of
`tensorflow\_probability/python/mcmc/random\_walk\_metropolis.py <https://github.com/tensorflow/probability/blob/master/tensorflow_probability/python/mcmc/random_walk_metropolis.py>`__:

.. code::

          new_state_fn: Python callable which takes a list of state parts and a
            seed; returns a same-type `list` of `Tensor`s, each being a perturbation
            of the input state parts. The perturbation distribution is assumed to be
            a symmetric distribution centered at the input state part.
            Default value: `None` which is mapped to
              `tfp.mcmc.random_walk_normal_fn()`.


Dynamic HMC
^^^^^^^^^^^

We love NUTS, or to be more precise Dynamic HMC with complex stopping
rules. This part is actually all done outside of Aesara, for NUTS, it
includes: the leapfrog, dual averaging, tunning of mass matrix and step
size, the tree building, sampler related statistics like divergence and
energy checking. We actually have an Aesara version of HMC, but it has never
been used, and has been removed from the main repository. It can still be
found in the `git history
<https://github.com/pymc-devs/pymc/pull/3734/commits/0fdae8207fd14f66635f3673ef267b2b8817aa68>`__,
though.

Variational Inference (VI)
~~~~~~~~~~~~~~~~~~~~~~~~~~

The design of the VI module takes a different approach than
MCMC - it has a functional design, and everything is done within Aesara
(i.e., Optimization and building the variational objective). The base
class of variational inference is
`pymc.variational.Inference <https://github.com/pymc-devs/pymc/blob/main/pymc/variational/inference.py>`__,
where it builds the objective function by calling:

.. code:: python

        ...
        self.objective = op(approx, **kwargs)(tf)
        ...

Where:

.. code::

        op     : Operator class
        approx : Approximation class or instance
        tf     : TestFunction instance
        kwargs : kwargs passed to :class:`Operator`

The design is inspired by the great work `Operator Variational
Inference <https://arxiv.org/abs/1610.09033>`__. ``Inference`` object is
a very high level of VI implementation. It uses primitives: Operator,
Approximation, and Test functions to combine them into single objective
function. Currently we do not care too much about the test function, it
is usually not required (and not implemented). The other primitives are
defined as base classes in `this
file <https://github.com/pymc-devs/pymc/blob/main/pymc/variational/opvi.py>`__.
We use inheritance to easily implement a broad class of VI methods
leaving a lot of flexibility for further extensions.

For example, consider ADVI. We know that in the high-level, we are
approximating the posterior in the latent space with a diagonal
Multivariate Gaussian. In another word, we are approximating each elements in
``model.free_RVs`` with a Gaussian. Below is what happen in the set up:

.. code:: python

    def __init__(self, *args, **kwargs):
        super(ADVI, self).__init__(MeanField(*args, **kwargs))
    # ==> In the super class KLqp
        super(KLqp, self).__init__(KL, MeanField(*args, **kwargs), None, beta=beta)
    # ==> In the super class Inference
        ...
        self.objective = KL(MeanField(*args, **kwargs))(None)
        ...

where ``KL`` is Operator based on Kullback Leibler Divergence (it does
not need any test function).

.. code:: python

        ...
        def apply(self, f):
            return -self.datalogp_norm + self.beta * (self.logq_norm - self.varlogp_norm)

Since the logp and logq are from the approximation, let's dive in
further on it (there is another abstraction here - ``Group`` - that
allows you to combine approximation into new approximation, but we will
skip this for now and only consider ``SingleGroupApproximation`` like
``MeanField``): The definition of ``datalogp_norm``, ``logq_norm``,
``varlogp_norm`` are in
`variational/opvi <https://github.com/pymc-devs/pymc/blob/main/pymc/variational/opvi.py>`__,
strip away the normalizing term, ``datalogp`` and ``varlogp`` are
expectation of the variational free\_RVs and data logp - we clone the
datalogp and varlogp from the model, replace its input with Aesara
tensor that `samples from the variational
posterior <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/variational/opvi.py#L1098-L1111>`__.
For ADVI, these samples are from `a
Gaussian <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/variational/approximations.py#L84-L89>`__.
Note that the samples from the posterior approximations are usually 1
dimension more, so that we can compute the expectation and get the
gradient of the expectation (by computing the `expectation of the
gradient! <http://blog.shakirm.com/2015/10/machine-learning-trick-of-the-day-4-reparameterisation-tricks/>`__).
As for the ``logq`` since it is a Gaussian `it is pretty
straightforward to evaluate <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/variational/approximations.py#L91-L97>`__.

Some challenges and insights from implementing VI.
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

-  Graph based approach was helpful, but Aesara had no direct access to
   previously created nodes in the computational graph. you can find a
   lot of ``@node_property`` usages in implementation. This is done to
   cache nodes. TensorFlow has graph utils for that that could
   potentially help in doing this. On the other hand graph management in
   Tensorflow seemed to more tricky than expected. The high level reason
   is that graph is an add only container

-  There were few fixed bugs not obvoius in the first place. Aesara has
   a tool to manipulate the graph (``aesara.clone_replace``) and this tool
   requires extremely careful treatment when doing a lot of graph
   replacements at different level.

-  We coined a term ``aesara.clone_replace`` curse. We got extremely dependent
   on this feature. Internal usages are uncountable:

   -  we use this to `vectorize the
      model <https://github.com/pymc-devs/pymc/blob/main/pymc/model.py#L972>`__
      for both MCMC and VI to speed up computations
   -  we use this to `create sampling
      graph <https://github.com/pymc-devs/pymc/blob/main/pymc/variational/opvi.py#L1483>`__
      for VI. This is the case you want posterior predictive as a part
      of computational graph.

As this is the core of the VI process, we were trying to replicate this pattern
in TF. However, when ``aesara.clone_replace`` is called, Aesara creates a new part of the graph that can
be collected by garbage collector, but TF's graph is add only. So we
should solve the problem of replacing input in a different way.

Forward sampling
----------------

As explained above, in distribution we have method to walk the model
dependence graph and generate forward random sample in scipy/numpy. This
allows us to do prior predictive samples using
``pymc.sampling.sample_prior_predictive`` see `code <https://github.com/pymc-devs/pymc/blob/6d07591962a6c135640a3c31903eba66b34e71d8/pymc/sampling.py#L1303-L1345>`__.
It is a fairly fast batch operation, but we have quite a lot of bugs and
edge case especially in high dimensions. The biggest pain point is the
automatic broadcasting. As in the batch random generation, we want to
generate (n\_sample, ) + RV.shape random samples. In some cases, where
we broadcast RV1 and RV2 to create a RV3 that has one more batch shape,
we get error (even worse, wrong answer with silent error).

The good news is, we are fixing these errors with the amazing works from [lucianopaz](https://github.com/lucianopaz) and
others. The challenge and some summary of the solution could be found in Luciano's [blog post](https://lucianopaz.github.io/2019/08/19/pymc-shape-handling/)

.. code:: python

    with pm.Model() as m:
        mu = pm.Normal('mu', 0., 1., shape=(5, 1))
        sd = pm.HalfNormal('sd', 5., shape=(1, 10))
        pm.Normal('x', mu=mu, sigma=sd, observed=np.random.randn(2, 5, 10))
        trace = pm.sample_prior_predictive(100)

    trace['x'].shape # ==> should be (100, 2, 5, 10)

.. code:: python

    pm.Normal.dist(mu=np.zeros(2), sigma=1).random(size=(10, 4))

There are also other error related random sample generation (e.g.,
`Mixture is currently
broken <https://github.com/pymc-devs/pymc/issues/3270>`__).

Extending PyMC
--------------

-  Custom Inference method
    -  `Inferencing Linear Mixed Model with EM.ipynb <https://github.com/junpenglao/Planet_Sakaar_Data_Science/blob/master/Ports/Inferencing%20Linear%20Mixed%20Model%20with%20EM.ipynb>`__
    -  `Laplace approximation in  pymc.ipynb <https://github.com/junpenglao/Planet_Sakaar_Data_Science/blob/master/Ports/Laplace%20approximation%20in%20pymc.ipynb>`__
-  Connecting it to other library within a model
    -  `Using “black box” likelihood function by creating a custom Aesara Op <https://docs.pymc.io/notebooks/blackbox_external_likelihood.html>`__
    -  Using emcee
-  Using other library for inference
    -  Connecting to Julia for solving ODE (with gradient for solution that can be used in NUTS)

What we got wrong
-----------------

Shape
~~~~~

One of the pain point we often face is the issue of shape. The approach
in TFP and pyro is currently much more rigorous. Adrian’s PR
(https://github.com/pymc-devs/pymc/pull/2833) might fix this problem,
but likely it is a huge effort of refactoring. I implemented quite a lot
of patches for mixture distribution, but still they are not done very
naturally.

Random methods in numpy
~~~~~~~~~~~~~~~~~~~~~~~

There is a lot of complex logic for sampling from random variables, and
because it is all in Python, we can't transform a sampling graph
further. Unfortunately, Aesara does not have code to sample from various
distributions and we didn't want to write that our own.

Samplers are in Python
~~~~~~~~~~~~~~~~~~~~~~

While having the samplers be written in Python allows for a lot of
flexibility and intuitive for experiment (writing e.g. NUTS in Aesara is
also very difficult), it comes at a performance penalty and makes
sampling on the GPU very inefficient because memory needs to be copied
for every logp evaluation.
