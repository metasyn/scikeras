.. _Migration:

=================================================
Migrating from ``tf.keras.wrappers.scikit_learn``
=================================================

.. contents::
   :local:


Why switch to SciKeras
----------------------
SciKeras has several advantages over ``tf.keras.wrappers.scikit_learn``:

* Full compatibility with the Scikit-Learn API, including grid searches, ensembles, transformers, etc.
* Support for Functional and Subclassed Keras Models.
* Support for pre-trained models.
* Support for dynamically set Keras parameters depending on inputs (e.g. input shape).
* Support for hyperparameter tuning of optimizers and losses.
* Support for multi-input and multi-ouput Keras models.
* Functional `random_state` for reproducible training.
* Many more that you will discover as you use SciKeras!


Changes to your code
--------------------

SciKeras is largely backwards compatible with the existing wrappers. For most cases, you can just change your import statement from:

.. code:: diff

   - from tensorflow.keras.wrappers.scikit_learn import KerasClassifier, KerasRegressor
   + from scikeras.wrappers import KerasClassifier, KerasRegressor


SciKeras does however have some backward incompatible changes:

Fit returns ``self``
^^^^^^^^^^^^^^^^^^

In TensorFlow calling ``fit`` on wrappers returned a Keras ``History`` object, which held an attribute called ``history`` that is
a dictionary with loss and metric names as keys and lists of recorded values for each epoch as the dictionary values.
However, in SciKeras, ``fit`` now returns and instance of the estimator itself in order to conform to the Scikit-Learn API.
Instead, the history is saved in the ``history_`` attribute.
Calling ``fit`` resets this attribute, calling ``partial_fit`` on the other hand extends it.

.. code:: diff

   def get_model():
      ...
      model.compile(loss="mse", metrics=["mae"])
      return model

   clf = KerasClassifier(get_model)
   - hist = clf.fit(...).history
   - losses = hist["mae"]
   + hist = clf.fit(...).history_
   + losses = hist["mean_absolute_error"]

.. note::
   Unlike the TensorFlow wrappers, SciKeras normalizes the names of the keys, so that if you use `metrics=["mae"]` you will get a key named `"mean_absolute_error"`.

``score()`` return R^2 by default for regressors
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Keras returns your model's mean loss when ``KerasRegressor.score()`` is called.
SciKeras, like most Scikit-Learn estimators, returns the coefficient of determination of the prediction (also known as R^2 score).
This is however customizable.

One-hot encoding of targets for categorical crossentropy losses
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SciKeras will not longer implicitly inspect your Model's loss function to determine if
it needs to one-hot encode your target to match the loss function. Instead, you must explicitly
pass your loss function to the constructor:

.. code:: python

   clf = KerasClassifier(loss="categorical_crossentropy")

Variable keyword arguments in fit and predict
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Keras supports a variable keyword arguments (commonly referred to as ``**kwargs``) for ``fit`` and ``predict``.
Scikit-Learn on the other hand does not support these arguments, and using them is largely incompatible with the Scikit-Learn ecosystem.
As a compromise, SciKeras supports these arguments, but we recommended that you set parameters using the constructor
or ``set_params`` for first-class SciKeras support.

For example, to declare ``batch_size`` in the constructor:

.. code:: diff

   - clf = KerasClassifier(...)
   - clf.fit(..., batch_size=32)
   + clf = KerasClassifier(..., batch_size=32)
   + clf.fit(...)

Or to declare separate values for ``fit`` and ``predict``:

.. code:: python

   clf = KerasClassifier(..., fit__batch_size=32, predict__batch_size=10000)

If you want to change the parameters on a live instance, you can do:

.. code:: python

   clf = KerasClassifier(...)
   clf.set_params(fit__batch_size=32, predict__batch_size=10000)
   clf.fit(...)

Functionally, this is the same as passing these parameters to ``fit``, just with one more function call.
This is much more compatible with the Scikit-Learn API.
In fact, this is what Scikti-Learn does in the background for hyperparameter tuning.

Renaming of ``build_fn`` to ``model``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SciKeras renamed the constructor argument ``build_fn`` to ``model``. In a future release,
passing ``build_fn`` as a _keyword_ argument will raise a ``TypeError``. Passing it as a positional
argument remains unchanged. You can make the following change to future proof your code:

.. code:: diff

   - clf = KerasClassifier(build_fn=...)
   + clf = KerasClassifier(model=...)

Default arguments in build_fn/model
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SciKeras will no longer introspect your callable `model` for *user defined* parameters
(the behavior for parameters like ``optimizer`` is unchanged). 
You must now "declare" them as keyword arguments to the constructor if you want them to be
tunable parameters (i.e. settable via ``set_params``):

.. code:: diff

   - def get_model(my_param=123):
   + def get_model(my_param):  # You can optionally remove the default here
      ...
      return model

   - clf = KerasClassifier(get_model)
   + clf = KerasClassifier(get_model, my_param=123)  # option 1
   + clf = KerasClassifier(get_model, model__my_param=123)  # option 2

That said, if you do not need them to work with ``set_params`` (which is only really
necessary if you are doing hyperparameter tuning), you do not need to make any changes.
