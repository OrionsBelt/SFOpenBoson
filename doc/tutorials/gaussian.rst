.. _tutorial_gaussian:

Forced quantum harmonic oscillator
======================================

.. sectionauthor:: Josh Izaac <josh@xanadu.ai>

Simulating the time-propagation of a Gaussian Hamiltonian using a continuous-variable (CV) quantum circuit is simple with SFOpenBoson. In this tutorial, we will walk through a simple example using the forced quantum harmonic oscillator.

Background
----------

The Hamiltonian of the forced quantum harmonic oscillator is given by 

.. math:: \hat{H} = \frac{\p^2}{2m} + \frac{1}{2}m\omega^2 \q^2 - F\q

where

* :math:`m` is the mass of the oscillator,
* :math:`\omega` is the frequency of oscillation,
* :math:`F` is a time-independent external force.

Let's define this Hamiltonian using OpenFermion, with :math:`m=\omega=1` and :math:`F=2`:

>>> from openfermion.ops import QuadOperator
>>> from openfermion.utils import commutator, normal_ordered
>>> H = QuadOperator('q0 q0', 0.5) + QuadOperator('p0 p0', 0.5) - QuadOperator('q0', 2)

In the Heisenberg picture, the time-evolution of the :math:`\q` and :math:`\p` operators is given by:

.. math::
	& \frac{d}{dt}\q = \frac{i}{\hbar}[\hat{H}, \q] =  \p\\
	& \frac{d}{dt}\p = \frac{i}{\hbar}[\hat{H}, \q] = F-\q

We can double check these using OpenFermion:

>>> (1j/2)*normal_ordered(commutator(H, QuadOperator('q0')), hbar=2)
1 [p0]
>>> (1j/2)*normal_ordered(commutator(H, QuadOperator('p0')), hbar=2)
2 [] + -1 [q0]

Assuming the oscillator has initial conditions :math:`\q(0)` and :math:`\p(0)`, it is easy to solve this coupled set of linear differential analytically, giving the parametrised solution

.. math::
	&\q(t) = (\q(0)-F)\cos(t) + \p(0)\sin(t) + F\\
	&\p(t) = (F-\q(0))\sin(t) + \p(0)\cos(t)

Let's now attempt to simulate these dynamics directly in Strawberry Fields, solely from the Hamiltonian we defined above.

Strawberry Fields simulation
----------------------------

To simulate the time-propagation of the forced oscillator in StrawberryFields, we also need to import the :class:`~.GaussianPropagation` class from the SFOpenBoson plugin:

>>> import strawberryfields as sf
>>> from strawberryfields.ops import *
>>> from sfopenboson.ops import GaussianPropagation

:class:`~.GaussianPropagation` accepts the following arguments:

* ``operator``: a bosonic Gaussian Hamiltonian, either in the form of a ``BosonOperator`` or ``QuadOperator``.

* ``t`` (float): the time propagation value. If not provided, default value is 1.

* ``mode`` (str): By default, ``mode='local'`` and the Hamiltonian is assumed to apply to only the applied qumodes. For example, if ``QuadOperator('q0 p1') | (q[2], q[4])``, then ``q0`` acts on ``q[2]``, and ``p1`` acts on ``q[4]``.

Alternatively, you can set ``mode='global'``, and the Hamiltonian is instead applied to the entire register by directly matching qumode numbers of the defined Hamiltonian; i.e., ``q0`` is applied to ``q[0]``, ``p1`` is applied to ``q[1]``, etc.

Let's set up the one qumode quantum circuit, propagating the forced oscillator Hamiltonian ``H`` we defined in the previous section, starting from the initial location :math:`(1,0.5)` in phase space, for time :math:`t=1.43`:

>>> eng, q = sf.Engine(1)
>>> with eng:
...     Xgate(1) | q[0]
...     Zgate(0.5) | q[0]
...     GaussianPropagation(H, 1.43) | q

Now, we can run this simulation using the `Gaussian backend of Strawberry Fields <https://strawberryfields.readthedocs.io/en/latest/code/backend.gaussian.html>`_, and output the location of the oscillator in phase space at time :math:`t=1.43`:

>>> state = eng.run('gaussian')
>>> state.means()
array([ 2.35472067,  1.06027036])

We compare this to the analytic solution,

.. math::
	&\braket{\q(1.43)} = (1-2)\cos(1.43) + 0.5\sin(1.43) + 2 = 2.35472,\\
	&\braket{\p(1.43)} = (2-1)\sin(1.43) + 0.5\cos(1.43) = 1.06027,

which is in good agreement with the Strawberry Fields result.

We can also print the CV gates applied by the engine, to see how our time-evolution operator :math:`e^{-i\hat{H}t/\hbar}` got decomposed:

>>> eng.print_applied()
Xgate(1), 	(reg[0])
Zgate(0.5), 	(reg[0])
Rgate(-1.43), 	(reg[0])
Xgate(1.719), 	(reg[0])
Zgate(1.98), 	(reg[0])


Plotting the phase-space time evolution
----------------------------------------

By looping over various values of :math:`t`, we can plot the phase space location of the oscillator for various values of :math:`t`.

Consider the following example:

.. code-block:: python

	eng, q = sf.Engine(1, hbar=2)

	t_vals = np.arange(0, 6, 0.02)
	results = np.zeros([2, len(t_vals)])

	for step, t in enumerate(t_vals):
	    eng.reset()
	    with eng:
	        Xgate(1) | q[0]
	        Zgate(0.5) | q[0]
	        GaussianPropagation(H, t) | q

	    state = eng.run('gaussian')
	    results[:, step] = state.means()

Here, we are looping over the same circuit as above for values of :math:`t` within the domain :math:`0\leq t<6`, and storing the resulting expectation values :math:`(\braket{\q(t)}, \braket{\p(t)})` in the array ``results``. We can plot this array in phase space:

>>> from matplotlib import pyplot as plt
>>> plt.plot(*results)

.. image:: ../_static/forced_qho.png
    :align: center
    :target: javascript:void(0);
