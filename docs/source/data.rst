Data
----

In Edward, data is represented as TensorFlow tensors or NumPy arrays.

.. code:: python

    x_data = np.array([0, 1, 0, 0, 0, 0, 0, 0, 0, 1])
    x_data = tf.constant([0, 1, 0, 0, 0, 0, 0, 0, 0, 1])


There are three ways to read data in Edward. They follow the `three ways
to read data in TensorFlow
<https://www.tensorflow.org/versions/master/how_tos/reading_data/index.html>`__.

1. **Preloaded data.** A constant or variable in the TensorFlow graph
   holds all the data.

   This setting is the fastest to work with and is recommended if the
   data fits in memory.

   Represent the data as NumPy arrays.
   Internally, during inference, we will store them in TensorFlow variables to prevent
   copying data more than once in memory.
   (As an example, see
   the `mixture of Gaussians
   <https://github.com/blei-lab/edward/blob/master/examples/tf_mixture_gaussian.py>`__.)

2. **Feeding.** Manual code provides the data when running each step of
   inference.

   This setting provides the most fine-grained control which is useful for experimentation.

   Represent the data as TensorFlow placeholders. During inference,
   the user must manually feed the placeholders at each
   step by first initializing via ``inference.initialize()``; then
   in a loop call ``inference.update(feed_dict={...})`` where
   ``feed_dict`` carries the values for the ``tf.placeholder``'s.
   (As an example, see
   the `Bayesian linear regression
   <https://github.com/blei-lab/edward/blob/master/examples/bayesian_linear_regression.py>`__.)

3. **Reading from files.** An input pipeline reads the data from files
   at the beginning of a TensorFlow graph.

   This setting is recommended if the data does not fit in memory.

   Represent the data as TensorFlow tensors, where the tensors are the
   output of data readers. During inference, each update will be
   automatically evaluated over new batch tensors represented through
   the data readers. (As an example, see
   the `data unit test
   <https://github.com/blei-lab/edward/blob/master/tests/test_inference_data.py>`__.)

Training Models with Data
^^^^^^^^^^^^^^^^^^^^^^^^^

To pass in data during inference, we form a Python dictionary. Each
item in the dictionary has a random variable binded to the data
values.

.. code:: python

    # assuming `x` and `y` form observed variables in the model
    data = {x: x_data, y: y_data}


How do we use the data during training? In general there are three use
cases:

1. Train over the full data per step.

   Follow the setting of preloaded data.

2. Train over a batch per step when the full data fits in memory. This
   scale inference in terms of computational complexity.

   Follow the setting of preloaded data. Specify the batch size with
   ``n_minibatch`` in ``Inference``. By default, we will subsample by
   slicing along the first dimension of every data structure in the
   data dictionary. Alternatively, follow the setting of feeding.
   Manually deal with the batch behavior at each training step.

3. Train over batches per step when the full data does not fit in
   memory. This scales inference in terms of computational complexity and
   memory complexity.

   Follow the setting of reading from files. Alternatively, follow the
   setting of feeding, and use a generator to create and destroy NumPy
   arrays on the fly for feeding the placeholders.

Training Model Wrappers with Data
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

During inference, data is passed in differently for external
languages which use a model wrapper. Instead of binding observed
variables, it is usually comprised
of strings binded to the values, such as a key ``'x'`` with value
``np.array([0.23512, 13.2])``.
We detail specifics for each external language below.

-  **TensorFlow.** The data carries whatever keys and values the user
   accesses in the user-defined model. Key is a string. Value is a NumPy
   array or TensorFlow tensor.

.. code:: python

  class BetaBernoulli:
    def log_prob(self, xs, zs):
      log_prior = beta.logpdf(zs['p'], a=1.0, b=1.0)
      log_lik = tf.reduce_sum(bernoulli.logpmf(xs['x'], p=zs['p']))
      return log_lik + log_prior

  model = BetaBernoulli()
  data = {'x': np.array([0, 1, 0, 0, 0, 0, 0, 0, 0, 1])}

-  **Python.** The data carries whatever keys and values the user
   accesses in the user-defined model. Key is a string. Value is a NumPy
   array or TensorFlow tensor.

.. code:: python

  class BetaBernoulli(PythonModel):
    def _py_log_prob(self, xs, zs):
      log_prior = beta.logpdf(zs['p'], a=1.0, b=1.0)
      log_lik = np.sum(bernoulli.logpmf(xs['x'], p=zs['p']))
      return log_lik + log_prior

  model = BetaBernoulli()
  data = {'x': np.array([0, 1, 0, 0, 0, 0, 0, 0, 0, 1])}

-  **PyMC3.** The data binds Theano shared variables, which are used to
   mark the observed PyMC3 random variables, to their realizations. Key
   is a Theano shared variable. Value is a NumPy array or TensorFlow
   tensor.

.. code:: python

  x_obs = theano.shared(np.zeros(1))
  with pm.Model() as pm_model:
    p = pm.Beta('p', 1, 1, transform=None)
    x = pm.Bernoulli('x', p, observed=x_obs)

  model = PyMC3Model(pm_model)
  data = {x_obs: np.array([0, 1, 0, 0, 0, 0, 0, 0, 0, 1])}

-  **Stan.** The data is according to the Stan program's data block. Key
   is a string. Value is whatever type is used for the data block.

.. code:: python

  model_code = """
    data {
      int<lower=0> N;
      int<lower=0,upper=1> x[N];
    }
    parameters {
      real<lower=0,upper=1> p;
    }
    model {
      p ~ beta(1.0, 1.0);
      for (n in 1:N)
      x[n] ~ bernoulli(p);
    }
  """
  model = ed.StanModel(model_code=model_code)
  data = {'N': 10, 'x': [0, 1, 0, 0, 0, 0, 0, 0, 0, 1]}

Note that for these model wrappers,
all 3 use cases for training models with data are supported. However,
Stan is limited to training over the full data per step. (This
because Stan's data structure requires data subsampling on arbitrary
data types, which we don't know how to automate.)
