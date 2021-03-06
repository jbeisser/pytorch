Autograd mechanics
==================

This note will present an overview of how autograd works and records the
operations. It's not strictly necessary to understand all this, but we recommend
getting familiar with it, as it will help you write more efficient, cleaner
programs, and can aid you in debugging.

.. _excluding-subgraphs:

Excluding subgraphs from backward
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Every Variable has two flags: :attr:`requires_grad` and :attr:`volatile`.
They both allow for fine grained exclusion of subgraphs from gradient
computation and can increase efficiency.

.. _excluding-requires_grad:

``requires_grad``
~~~~~~~~~~~~~~~~~

If there's a single input to an operation that requires gradient, its output
will also require gradient. Conversely, only if all inputs don't require
gradient, the output also won't require it. Backward computation is never
performed in the subgraphs, where all Variables didn't require gradients.

.. code::

    >>> x = Variable(torch.randn(5, 5))
    >>> y = Variable(torch.randn(5, 5))
    >>> z = Variable(torch.randn(5, 5), requires_grad=True)
    >>> a = x + z
    >>> a.requires_grad
    False
    >>> b = a + z
    >>> b.requires_grad
    True

This is especially useful when you want to freeze part of your model, or you
know in advance that you're not going to use gradients w.r.t. some parameters.
For example if you want to finetune a pretrained CNN, it's enough to switch the
:attr:`requires_grad` flags in the frozen base, and no intermediate buffers will
be saved, until the computation gets to the last layer, where the affine
transform will use weights that require gradient, and the output of the network
will also require them.

.. code::

    model = torchvision.models.resnet18(pretrained=True)
    for param in model.parameters():
        param.requires_grad = False
    # Replace the last fully-connected layer
    # Parameters of newly constructed modules have requires_grad=True by default
    model.fc = nn.Linear(512, 100)

    # Optimize only the classifier
    optimizer = optim.SGD(model.fc.parameters(), lr=1e-2, momentum=0.9)

``volatile``
~~~~~~~~~~~~

Volatile is recommended for purely inference mode, when you're sure you won't
be even calling `.backward()`. It's more efficient than any other autograd
setting - it will use the absolute minimal amount of memory to evaluate the
model. ``volatile`` also determines that ``requires_grad is False``.

Volatile differs from :ref:`excluding-requires_grad` in how the flag propagates.
If there's even a single volatile input to an operation, its output is also
going to be volatile. Volatility spreads accross the graph much easier than
non-requiring gradient - you only need a **single** volatile leaf to have a
volatile output, while you need **all** leaves to not require gradient to
have an output the doesn't require gradient. Using volatile flag you don't
need to change any settings of your model parameters to use it for
inference. It's enough to create a volatile input, and this will ensure that
no intermediate states are saved.

.. code::

    >>> regular_input = Variable(torch.randn(5, 5))
    >>> volatile_input = Variable(torch.randn(5, 5), volatile=True)
    >>> model = torchvision.models.resnet18(pretrained=True)
    >>> model(regular_input).requires_grad
    True
    >>> model(volatile_input).requires_grad
    False
    >>> model(volatile_input).volatile
    True
    >>> model(volatile_input).creator is None
    True

How autograd encodes the history
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each Variable has a ``.creator`` attribute, that points to the function, of
which it is an output. This is an entry point to a directed acyclic graph (DAG)
consisting of :class:`Function` objects as nodes, and references between them
being the edges. Every time an operation is performed, a new :class:`Function`
representing it is instantiated, its :meth:`~torch.autograd.Function.forward`
method is called, and its output :class:`Variable` s creators are set to it.
Then, by following the path from any :class:`Variable` to the leaves, it is
possible to reconstruct the sequence of operations that has created the data,
and automatically compute the gradients.

An important thing to note is that the graph is recreated from scratch at every
iteration, and this is exactly what allows for using arbitrary Python control
flow statements, that can change the overall shape and size of the graph at
every iteration. You don't have to encode all possible paths before you
launch the training - what you run is what you differentiate.

In-place operations on Variables
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Supporting in-place operations in autograd is a hard matter, and we discourage
their use in most cases. Autograd's aggressive buffer freeing and reuse makes
it very efficient and there are very few occasions when in-place operations
actually lower memory usage by any significant amount. Unless you're operating
under heavy memory pressure, you might never need to use them.

There are two main reasons that limit the applicability of in-place operations:

1. Overwriting values required to compute gradients. This is why variables don't
   support ``log_``. Its gradient formula requires the original input, and while
   it is possible to recreate it by computing the inverse operation, it is
   numerically unstable, and requires additional work that often defeats the
   purpose of using these functions.

2. Every in-place operation actually requires the implementation to rewrite the
   computational graph. Out-of-place versions simply allocate new objects and
   keep references to the old graph, while in-place operations, require
   changing the creator of all inputs to the :class:`Function` representing
   this operation. This can be tricky, especially if there are many Variables
   that reference the same storage (e.g. created by indexing or transposing),
   and in-place functions will actually raise an error if the storage of
   modified inputs is referenced by any other :class:`Variable`.

In-place correctness checks
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Every variable keeps a version counter, that is incremented every time it's
marked dirty in any operation. When a Function saves any tensors for backward,
a version counter of their containing Variable is saved as well. Once you access
``self.saved_tensors`` it is checked, and if it's greater than the saved value
an error is raised.
