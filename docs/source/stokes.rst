.. Automatically generated Sphinx-extended reStructuredText file from DocOnce source
   (https://github.com/hplgit/doconce/)

.. Document title:

Demo - Stokes equations
%%%%%%%%%%%%%%%%%%%%%%%

:Authors: Mikael Mortensen (mikaem at math.uio.no)
:Date: Feb 8, 2019

*Summary.* The Stokes equations describe the flow of highly viscous fluids.
This is a demonstration of how the Python module `shenfun <https://github.com/spectralDNS/shenfun>`__ can be used to solve Stokes
equations using a  mixed (coupled) basis in a 3D tensor product domain.
We assume homogeneous Dirichlet boundary conditions in one direction
and periodicity in the remaining two. The solver described runs with MPI
without any further considerations required from the user.
The solver assembles a block matrix with sparsity pattern as shown below
for the Legendre basis.

.. _fig:BlockMat:

.. figure:: https://rawgit.com/spectralDNS/spectralutilities/master/figures/BlockMat.png

   *Coupled block matrix for Stokes equations*

Model problem
=============

.. _demo:stokes:

Stokes equations
----------------

The Stokes equations are given in strong form as

.. math::
        \begin{align*}
        \nabla^2 \boldsymbol{u} - \nabla p &= \boldsymbol{f} \quad \text{in }  \Omega, \\ 
        \nabla \cdot \boldsymbol{u} &= h \quad \text{in } \Omega  \\ 
        \int_{\Omega} p dx &= 0
        \end{align*}

where :math:`\boldsymbol{u}` and :math:`p` are, respectively, the
fluid velocity vector and pressure, and the domain
:math:`\Omega = [0, 2\pi]^2 \times [-1, 1]`. The flow is assumed periodic
in :math:`x` and :math:`y`-directions, whereas there is a no-slip homogeneous Dirichlet
boundary condition on :math:`\boldsymbol{u}` on the boundaries of the :math:`z`-direction, i.e.,
:math:`\boldsymbol{u}(x, y, \pm 1) = (0, 0, 0)`. (Note that we can configure shenfun with
non-periodicity in any of the three directions. However, since we are to
solve linear algebraic systems in the non-periodic direction, there is a speed
benefit from having the nonperiodic direction last. This has to do with Numpy
using a C-style row-major storage of arrays by default.)
The right hand side vector :math:`\boldsymbol{f}(\boldsymbol{x})` is an external applied body force.
The right hand side :math:`h` is usually zero in the regular Stokes equations. Here
we include it because it will be nonzero in the verification, which is using the
method of manufactured solutions. Note that the final :math:`\int_{\Omega} p dx = 0`
is there because there is no Dirichlet boundary condition on the pressure
and the system of equations would otherwise be ill conditioned.

To solve Stokes equations with the Galerkin method we need basis
functions for both velocity and pressure. A
Dirichlet basis will be used for velocity, whereas there is no boundary restriction
on the pressure basis. For both three-dimensional bases we will use one basis
function for the :math:`x`-direction,
:math:`\mathcal{X}(x)`, one for the :math:`y`-direction, :math:`\mathcal{Y}(y)`, and one for the
:math:`z`-direction, :math:`\mathcal{Z}(z)`. And
then we create three-dimensional basis functions like

.. math::
   :label: _auto1

        
        v(x, y, z) = \mathcal{X}(x) \mathcal{Y}(y) \mathcal{Z} (z).
        
        

The basis functions :math:`\mathcal{X}(x)` and :math:`\mathcal{Y}(y)` are chosen as Fourier
exponentials, since these functions are periodic:

.. math::
   :label: _auto2

        
        \mathcal{X}_l(x) = e^{\imath l x}, \forall \, l \in \boldsymbol{l}^{N_0}, 
        
        

.. math::
   :label: _auto3

          
        \mathcal{Y}_m(y) =  e^{\imath m y}, \forall \, m \in \boldsymbol{m}^{N_1},
        
        

where :math:`\boldsymbol{l}^{N_0} = (-N_0/2, -N_0/2+1, \ldots, N_0/2-1)` and
:math:`\boldsymbol{m}^{N_1} = (-N_1/2, -N_1/2+1, \ldots, N_1/2-1)`.
The size of the discretized problem in real physical space is
:math:`\boldsymbol{N} = (N_0, N_1, N_2)`, i.e., there are :math:`N_0 \cdot N_1 \cdot N_2` quadrature points
in total.

The basis functions for :math:`\mathcal{Z}(z)` remain to be decided.
For the velocity we need homogeneous Dirichlet boundary conditions, and for this
we use composite Legendre or Chebyshev polynomials

.. math::
   :label: _auto4

        
        \mathcal{Z}^0_n(z) = \phi_n(z) - \phi_{n+2}(z), \forall \, n \in \boldsymbol{n}^{N_2},
        
        

where :math:`\phi_n` is the n'th Legendre or Chebyshev polynomial of the first kind.
:math:`\boldsymbol{n}^{N_2} = (0, 1, \ldots, N_2-3)`, and the zero on :math:`\mathcal{Z}^0`
is there to indicate the zero value on the boundary.

The pressure basis that comes with no restrictions for the boundary is a
little trickier. The reason for this has to do with
inf-sup stability. The obvious choice of basis is the regular Legendre or
Chebyshev basis, which is denoted as

.. math::
   :label: eq:Zn

        
        \mathcal{Z}_n(z) = \phi_n(z). 
        

The problem is that for the natural choice of :math:`n \in (0, 1, \ldots, N_2-1)`
there is a nullspace and one degree of freedom remains unresolved. It turns out
that the proper choice for the pressure basis is simply :eq:`eq:Zn` for
:math:`n \in \boldsymbol{n}^{N-2}`. (Also remember that we have to fix :math:`\int_{\Omega} p dx = 0`.)

With given basis functions we obtain the bases

.. math::
   :label: _auto5

        
        V^{N_0} = \text{span}\{ \mathcal{X}_l \}_{l\in\boldsymbol{l}^{N_0}}, 
        
        

.. math::
   :label: _auto6

          
        V^{N_1} = \text{span}\{ \mathcal{Y}_m \}_{m\in\boldsymbol{m}^{N_1}}, 
        
        

.. math::
   :label: _auto7

          
        V^{N_2} = \text{span}\{ \mathcal{Z}_n \}_{n\in\boldsymbol{n}^{N_2}}, 
        
        

.. math::
   :label: _auto8

          
        V_0^{N_2} = \text{span}\{ \mathcal{Z}^0_n \}_{n\in\boldsymbol{n}^{N_2}},
        
        

and from these we create two different tensor product spaces

.. math::
   :label: _auto9

        
        W_0^{\boldsymbol{N}}(\boldsymbol{x}) = V^{N_0}(x) \times V^{N_1}(y) \times V_0^{N_2}(z), 
        
        

.. math::
   :label: _auto10

          
        W^{\boldsymbol{N}}(\boldsymbol{x}) = V^{N_0}(x) \times V^{N_1}(y) \times V^{N_2}(z).
        
        

The velocity vector is using a mixed basis, such that we will look for
solutions :math:`\boldsymbol{u} \in [W_0^{\boldsymbol{N}}]^3`, whereas we look for the pressure
:math:`p \in W^{\boldsymbol{N}}`. We now formulate a variational problem using the Galerkin method: Find
:math:`\boldsymbol{u} \in [W_0^{\boldsymbol{N}}]^3` and :math:`p \in W^{\boldsymbol{N}}` such that

.. math::
   :label: eq:varform

        
        \int_{\Omega} (\nabla^2 \boldsymbol{u} - \nabla p ) \cdot \overline{\boldsymbol{v}} \, dx_w = \int_{\Omega} \boldsymbol{f} \cdot \overline{\boldsymbol{v}}\, dx_w \quad\forall \boldsymbol{v} \, \in \, [W_0^{\boldsymbol{N}}]^3,  
        

.. math::
   :label: _auto11

          
        \int_{\Omega} \nabla \cdot \boldsymbol{u} \, \overline{q} \, dx_w = \int_{\Omega} h \overline{q} \, dx_w \quad\forall q \, \in \, W^{\boldsymbol{N}}.
        
        

Here :math:`dx_w=w_xdxw_ydyw_zdz` represents a weighted measure, with weights :math:`w_x(x), w_y(y), w_z(z)`.
Note that it is only Chebyshev polynomials that
make use of a non-constant weight :math:`w_x=1/\sqrt{1-x^2}`. The Fourier weights are :math:`w_y=w_z=1/(2\pi)`
and the Legendre weight is :math:`w_x=1`.
The overline in :math:`\boldsymbol{\overline{v}}` and :math:`\overline{q}` represents a complex conjugate, which is needed here because
the Fourier exponentials are complex functions.

.. _sec:mixedform:

Mixed variational form
----------------------

Since we are to solve for :math:`\boldsymbol{u}` and :math:`p` at the same time, we formulate a
mixed (coupled) problem: find :math:`(\boldsymbol{u}, p) \in [W_0^{\boldsymbol{N}}]^3 \times W^{\boldsymbol{N}}`
such that

.. math::
   :label: _auto12

        
        a((\boldsymbol{u}, p), (\boldsymbol{v}, q)) = L((\boldsymbol{v}, q)) \quad \forall (\boldsymbol{v}, q) \in [W_0^{\boldsymbol{N}}]^3 \times W^{\boldsymbol{N}},
        
        

where bilinear (:math:`a`) and linear (:math:`L`) forms are given as

.. math::
   :label: _auto13

        
            a((\boldsymbol{u}, p), (\boldsymbol{v}, q)) = \int_{\Omega} (\nabla^2 \boldsymbol{u} - \nabla p) \cdot \overline{\boldsymbol{v}} \, dx_w + \int_{\Omega} \nabla \cdot \boldsymbol{u} \, \overline{q} \, dx_w, 
        
        

.. math::
   :label: _auto14

          
            L((\boldsymbol{v}, q)) = \int_{\Omega} \boldsymbol{f} \cdot \overline{\boldsymbol{v}}\, dx_w + \int_{\Omega} h \overline{q} \, dx_w.
        
        

Note that the bilinear form will assemble to block matrices, whereas the right hand side
linear form will assemble to block vectors.

Implementation
==============

Preamble
--------

We will solve the Stokes equations using the `shenfun <https://github.com/spectralDNS/shenfun>`__ Python module. The first thing needed
is then to import some of this module's functionality
plus some other helper modules, like `Numpy <https://numpy.org>`__ and `Sympy <https://sympy.org>`__:

.. code-block:: python

    import os
    import sys
    import numpy as np
    from mpi4py import MPI
    from sympy import symbols, sin, cos, lambdify
    from shenfun import *

We use ``Sympy`` for the manufactured solution and ``Numpy`` for testing. MPI for
Python (``mpi4py``) is required for running the solver with MPI.

.. _sec:mansol:

Manufactured solution
---------------------

The exact solutions :math:`\boldsymbol{u}_e(\boldsymbol{x})` and :math:`p(\boldsymbol{x})` are chosen to satisfy boundary
conditions, and the right hand sides :math:`\boldsymbol{f}(\boldsymbol{x})` and :math:`h(\boldsymbol{x})` are then
computed exactly using ``Sympy``. These exact right hand sides will then be used to
compute a numerical solution that can be verified against the manufactured
solution. The chosen solution with computed right hand sides are:

.. code-block:: python

    uex = sin(2*y)*(1-z**2)
    uey = sin(2*x)*(1-z**2)
    uez = sin(2*z)*(1-z**2)
    pe = -0.1*sin(2*x)*cos(4*y)
    fx = uex.diff(x, 2) + uex.diff(y, 2) + uex.diff(z, 2) - pe.diff(x, 1)
    fy = uey.diff(x, 2) + uey.diff(y, 2) + uey.diff(z, 2) - pe.diff(y, 1)
    fz = uez.diff(x, 2) + uez.diff(y, 2) + uez.diff(z, 2) - pe.diff(z, 1)
    h = uex.diff(x, 1) + uey.diff(y, 1) + uez.diff(z, 1)
    
    # Lambdify for faster evaluation
    ulx = lambdify((x, y, z), uex, 'numpy')
    uly = lambdify((x, y, z), uey, 'numpy')
    ulz = lambdify((x, y, z), uez, 'numpy')
    flx = lambdify((x, y, z), fx, 'numpy')
    fly = lambdify((x, y, z), fy, 'numpy')
    flz = lambdify((x, y, z), fz, 'numpy')
    hl  = lambdify((x, y, z), h, 'numpy')
    pl  = lambdify((x, y, z), pe, 'numpy')

Bases and tensor product spaces
-------------------------------

Bases are created using the :func:`.Basis` function. The choice between
Legendre or Chebyshev is collected from the commandline, with Legendre as default

.. code-block:: text

    N = (40, 40, 40)
    family = sys.argv[-1] if len(sys.argv) == 2 else 'Legendre'
    K0 = Basis(N[0], 'Fourier', dtype='D', domain=(0, 2*np.pi))
    K1 = Basis(N[1], 'Fourier', dtype='d', domain=(0, 2*np.pi))
    SD = Basis(N[2], family, bc=(0, 0))
    ST = Basis(N[2], family)
    ST.slice = lambda: slice(0, ST.N-2)

Note that the last line of code is there to ensure that only the first
:math:`N_2-2` coefficients are used, see discussion around Eq. :eq:`eq:Zn`.
At the same time, we ensure that we are still using :math:`N_2`
quadrature points, the same as for the Dirichlet basis.

Next the bases are used to create two tensor product spaces WU = :math:`W^{\boldsymbol{N}}`
and WP = :math:`W_0^{\boldsymbol{N}}`, one vector UV = :math:`[W_0^{\boldsymbol{N}}]^3` and one mixed
space  Q = UV :math:`\times` WP.

.. code-block:: text

    WU = TensorProductSpace(comm, (K0, K1, SD), axes=(2, 0, 1))
    WP = TensorProductSpace(comm, (K0, K1, ST), axes=(2, 0, 1))
    UV = VectorTensorProductSpace(WU)
    Q = MixedTensorProductSpace([UV, WP])

Note that we choose to transform axes in the order :math:`1, 0, 2`. This is to ensure
that the fully transformed arrays are aligned in the non-periodic direction 2.
And we need the arrays aligned in this direction, because this is the only
direction where there are tensor product matrices that are non-diagonal. All
Fourier matrices are, naturally, diagonal.

Test- and trialfunctions are created much like in a regular, non-mixed,
formulation. However, one has to create one test- and trialfunction for
the mixed space, and then split them up afterwards

.. code-block:: text

    up = TrialFunction(Q)
    vq = TestFunction(Q)
    u, p = up
    v, q = vq

With the basisfunctions in place we may assemble the different blocks of the
final coefficient matrix. Since Legendre is using a constant weight function,
the equations may also be integrated by parts to obtain a symmetric system:

.. code-block:: text

    if family.lower() == 'chebyshev':
        A = inner(v, div(grad(u)))
        G = inner(v, -grad(p))
    else:
        A = inner(grad(v), -grad(u))
        G = inner(div(v), p)
    D = inner(q, div(u))

Here ``A, G`` and ``D`` are lists containg the different blocks in
the complete, coupled matrix. ``A`` actually contains 6
tensor product matrices of type :class:`.TPMatrix`. The first two
matrices are for vector component zero of the test function ``v[0]`` and
trial function ``u[0]``, the
matrices 2 and 3 are for components 1 and the last two are for components
2. The first two matrices are as such for

.. code-block:: text

      A[0:2] = inner(v[0], div(grad(u[0])))

Breaking it down the inner product is mathematically

.. math::
   :label: eq:partialeq1

        
        
        \int_{\Omega} \boldsymbol{v}[0] \left(\frac{\partial^2 \boldsymbol{u}[0]}{\partial x^2} + \frac{\partial^2 \boldsymbol{u}[0]}{\partial y^2} + \frac{\partial^2 \boldsymbol{u}[0]}{\partial z^2}\right) w_x dx w_y dy w_z dz.
        

If we now use test function :math:`\boldsymbol{v}[0]`

.. math::
   :label: _auto15

        
        \boldsymbol{v}[0]_{lmn} = \mathcal{X}_l \mathcal{Y}_m \mathcal{Z}_n,
        
        

and trialfunction

.. math::
   :label: _auto16

        
        \boldsymbol{u}[0]_{pqr} = \sum_{p} \sum_{q} \sum_{r} \hat{\boldsymbol{u}}[0]_{pqr} \mathcal{X}_p \mathcal{Y}_q \mathcal{Z}_r,
        
        

where :math:`\hat{\boldsymbol{u}}` are the unknown degrees of freedom, and then insert these functions
into :eq:`eq:partialeq1`, then we obtain after
performing some exact evaluations over the periodic directions

.. math::
   :label: _auto17

        
         \Big( \underbrace{-\left(l^2 \delta_{lp} + m^2 \delta_{mq} \right) \int_{-1}^{1} \mathcal{Z}_r(z) \mathcal{Z}_n(z) w_z dz}_{A[0]} + \underbrace{\delta_{lp} \delta_{mq} \int_{-1}^{1} \frac{\partial^2 \mathcal{Z}_r(z)}{\partial z^2} \mathcal{Z}_n(z) w_z dz}_{A[1]} \Big) \hat{\boldsymbol{u}}[0]_{pqr},
        
        

Similarly for components 1 and 2 of the test and trial vectors, leading to 6 tensor
product matrices in total for ``A``. Similarly, we get three components of ``G``
and  three of ``D``.

Eliminating the Fourier diagonal matrices, we are left with block matrices like

.. math::
   :label: eq:bmatrix

        H(l, m) =
          \begin{bmatrix}
              A[0]+A[1] & 0 & 0 & G[0] \\ 
              0 & A[2]+A[3] & 0 & G[1] \\ 
              0 & 0 &  A|[4]+A[5] & G[2] \\ 
              D[0] & D[1] & D[2] & 0
          \end{bmatrix}

Note that there will be one large block matrix :math:`H(l, m)` for each Fourier
wavenumber combination :math:`(l, m)`. To solve the problem in the end we will need to
loop over these wavenumbers and solve the assembled linear systems one by one.
An example of the block matrix, for :math:`l=m=5` and :math:`\boldsymbol{N}=(20, 20, 20)` is given
in Fig. :ref:`fig:BlockMat`.

There is still one more degree of freedom to take care of due to
:math:`\int_{\Omega} p dx = 0`. Due to all basis functions integrating to
zero, except from the zeroth, this simply boils down to setting
the unknown :math:`\hat{p}_{000}`, when the pressure trial function is
represented as

.. math::
         p(\boldsymbol{x}) = \sum_{l\in \boldsymbol{l}^{N_0}} \sum_{m\in \boldsymbol{m}^{N_1}} \sum_{n\in \boldsymbol{n}^{N_2}} \hat{p}_{lmn} \mathcal{X}_l \mathcal{Y}_m \mathcal{Z}_n.

So it is only necessary to fix one degree of freedom. This would
normally be a very easy procedure. Just indent the row of the
coefficient matrix :math:`H(0, 0)` corresponding to :math:`\hat{p}_{000}` and set
the corresponding right hand side to zero as well.
For the block matrix it is a bit complicated, because the indent
is part of the block (3, 3), which at the current time is zero,
see :eq:`eq:bmatrix`. Furthermore, it should only be enabled for
:math:`(l, m) = (0, 0)`.  For the Legendre basis it turns out to be
not very difficult. Just create a diagonal matrix :math:`I_{ij}` in the (3, 3)
block of :eq:`eq:bmatrix` with :math:`I_{00}=1` and zero otherwise, and then
add :math:`I_{ij}` to the (3, 3) block of :math:`H(0, 0)`. This can be coded as

.. code-block:: text

    P = inner(p, q) # assemble a diagonal matrix in block (3, 3)
    s = TD.shape(True)
    P.scale = np.zeros((s[0], s[1], 1))
    ls = TD.local_slice(True)
    if ls[0].start == 0 and ls[1].start == 0:   # enable only for Fourier l=m=0, which only lives on one processor
        P.scale[0, 0] = 1
    I = P.mats[2]
    I[0][:] = 0      # Zero the matrix diagonal (the only diagonal)
    I[0][0] = 1      # I_{0,0} = 1

For the Chebyshev basis it is a bit more complicated because the ``D[2]``
block of :eq:`eq:bmatrix` is not zero for the row of the :math:`\hat{p}_{000}`
degree of freedom. So we also need to do more work to do a proper indent

.. code-block:: text

    if family.lower() == 'chebyshev':
        # Have to ident global row (N[0]-2)*(N[1]-2)*(N[2]-2), but only for l=m=0.
        # This is a bit tricky.
        # For Legendre this row is already zero. With Chebyshev we need to modify
        # block (3, 2) as well as fixing the 1 on the diagonal of (3, 3)
        a0 = inner(q, div(u))[2]   # This TPMatrix will be used for only l=m=0
        a0.scale = np.zeros((s[0], s[1], 1))
        D[2].scale = np.ones((s[0], s[1], 1))
        if ls[0].start == 0 and ls[1].start == 0:
            a10.scale[0, 0] = 1     # enable for l=m=0
            D[2].scale[0, 0] = 0    # disable for l=m=0
        am = a0.pmat.diags().toarray()
        am[0] = 0   # Zero the row corresponding to p_hat[0, 0, 0]
        a0.mats[2] = extract_diagonal_matrix(am)
        a0.pmat = a0.mats[2]
        D.append(a0)

Note that ``a0`` is an instance of the :class:`.TPMatrix`, and scale is
an array of ndim = 3. The scale in indexed by the Fourier wavenumbers
and scale[0, 0] represents the scale to matrix ``a0`` for wavenumbers
:math:`(l, m) = (0, 0)`.

In the end we create a block matrix through

.. code-block:: text

    P = [P]
    M = BlockMatrix(A+G+D+P)

The right hand side can easily be assembled since we have already
defined the functions :math:`\boldsymbol{f}` and :math:`h`, see Sec. :ref:`sec:mansol`

.. code-block:: text

    # Get mesh (quadrature points)
    X = TD.local_mesh(True)
    
    # Get f and h on quad points
    fh = Array(Q)
    fh[0] = flx(*X)
    fh[1] = fly(*X)
    fh[2] = flz(*X)
    fh[3] = hl(*X)
    
    # Compute inner products
    fh_hat = Function(Q)
    fh_hat[:3] = inner(v, fh[:3], output_array=fh_hat[:3])
    fh_hat[3] = inner(q, fh[3], output_array=fh_hat[3])
    
    # Fix the right hand side of p_hat[0, 0, 0]
    fh_hat[3, 0, 0, 0] = 0

In the end all that is left is to solve and compare with
the exact solution

.. code-block:: text

    # Solve problem
    up_hat = M.solve(fh_hat)
    up = up_hat.backward()
    
    # Exact solution
    ux = ulx(*X)
    uy = uly(*X)
    uz = ulz(*X)
    pe = pl(*X)
    
    error = [comm.reduce(np.linalg.norm(ux-up[0])),
             comm.reduce(np.linalg.norm(uy-up[1])),
             comm.reduce(np.linalg.norm(uz-up[2])),
             comm.reduce(np.linalg.norm(pe-up[3]))]

.. _sec:3d:complete:

Complete solver
---------------

A complete solver can be found in demo `Stokes3D.py <https://github.com/spectralDNS/shenfun/blob/master/demo/Stokes3D.py>`__.
