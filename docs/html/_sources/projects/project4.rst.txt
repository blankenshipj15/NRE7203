.. _proj4:

Nodal Expansion Method and Discontinuity Factors
---------------------------------------------------------------

.. _Introduction:

=====================
Introduction
=====================

Lattice calculations for heterogeneous fuel assemblies tend to be computationally expensive. If multiphysics informed solutions are considered, the computation time can be prohibitive for design iteration.
Alternatives to using the high-fidelity Monte Carlo solution are nodal diffusion methods which are capable of developing quick and relatively accurate results. The heterogeneous assembly configuration is replaced with a homogenized region and the node is solved via nodal diffusion methods.
In particular, the Nodal Expansion Method is used to calculate homogenized group fluxes which are presented against the reference heterogeneous solution.
Taking a node *i* with a volume :math:`V^{i}=\Delta x^{i} \Delta y^{i} \Delta z^{i}`, the multigroup diffusion equation is given by: 

.. math::

  \nabla\cdot\mathbf{J}_{g}^{i}(\mathbf{r})+ [\Sigma_{ra,g}^{i}+\Sigma_{sp,g}^{i}]\phi_{g}^{i}({\bf r})=\sum_{g^{\prime}=1}^{G}\Sigma_{s p,g^{\prime}g}^{i}\phi_{g^{\prime}}^{i}({\bf r})+\frac{1}{k}\chi_{g}^{i}\sum_{g^{\prime}=1}^{G}\nu\Sigma_{f,g^{\prime}}^{i}\phi_{g^{\prime}}^{i}({\bf r})

The following matrices are defined.

.. math::

  \begin{gather}
  A_{g}^{i} = \Sigma_{ra,g}^{i}+\Sigma_{sp,g}^{i} \\
  Q_{g}^{i}(\mathbf{r}) = \sum_{g^{\prime}=1}^{G}\Sigma_{s p,g^{\prime}g}^{i}\phi_{g^{\prime}}^{i}(\mathbf{r})+\frac{1}{k}\chi_{g}^{i}\sum_{g^{\prime}=1}^{G}\nu\Sigma_{f,g^{\prime}}^{i}\phi_{g^{\prime}}^{i}(\mathbf{r})
  \end{gather}

The problems considered for this project are 2D with heterogenous solution data being provided by Serpent for the following assemblies.

.. figure:: NEM_DF_files/NEM_DF_6_0.png
  :width: 600
  :align: center

  SMR colorset.

The above assemblies are: Upper left **F12** - 2.6% :math:`UO_2` - Upper right **Ref** - stainless steel - Bottom left **F2** - 4.55% :math:`UO_2` with 8% :math:`Gd_2` :math:`O_3` - Bottom right **F11** - 2.6% :math:`UO_2`. 
Using NEM and transverse leakage methods, a 1D diffusion equation integrated in the transverse direction including the leakage term transverse to the solution direction is developed for node *i*:

.. math::

  \frac{\partial}{\partial y}\bar{J}_{gy}(y) + A_{g}\bar{\phi}_{gy}(y) = \bar{Q}_{gy}(y)-\frac{1}{\Delta x}\bar{L}_{gx}(y)

Fick's law is used to represent the current. For the nodal expansion method, it is assumed the nodal homogeneous flux can be expanded over a set of basis functions (PARCS defined basis functions are used) such that:

.. math::

  \bar{\phi}_{gy}^{i}(y) = \sum_{n=0}^{4}a_{gyn}^{i}f_n(y)

This expansion produces 4xG unknown coefficients and an unknown flux for each solution direction which are solved for simultaneously.

The transverse leakage term is approximated using a quadratic expression based on basis functions found in PARCS. The approximation is provided below.


.. math::

  \begin{gather}
  \bar{L}_{x}^{i}(y)=\bar{L}_{x}^{i}+\ell_{x1}^{i}f_{1}(y)+\ell_{x2}^{i}f_{2}(y) \\\\
  f_{1}(y)=\frac{y}{\Delta y^{i}} \\\\
  f_{2}(y)=3\left(\frac{y}{\Delta y^{i}}\right)^{2}-\frac{1}{4}
  \end{gather}

The coefficients of the quadratic are solved via a polynomial interpolation.
Finally, a weighted residual scheme is applied, producing a first and second moment of the base diffusion equation. The first and second moment equations are weighted with :math:`f_{1}` and :math:`f_{2}`, respectively.
With a set of conservation equations (average flux, net surface currents, weighted residuals) and appropriate boundary conditions supplied by the heterogeneous solution from Serpent, the 1D homogeneous flux distribution is obtained simultaneously through

.. math::

  \mathbf{A}\mathbf{x}=\mathbf{B}

where **x** contains the unknown expansion coefficients used to recover the homogenized flux and **B** contains flux, currents, and transverse leakages.

It is expected that the homogeneous flux will likely not be continuous at the node interface. Therefore, groupwise discontinuity factors (DF) are defined at each node interface to couple the homogeneous flux solution to the reference heterogenous flux solution.
This is accomplished by defining the DF as:

.. math::

  \begin{gather}
  f_{g}^{-}\equiv\frac{\phi_{g}^{h e t-}}{\phi_{g}^{h o m-}} \\\\
  f_{g}^{+}\equiv\frac{\phi_{g}^{h e t+}}{\phi_{g}^{h o m+}}
  \end{gather}

where the + and - indicate the north (+) and south (-) node faces. The heterogeneous flux is provided as a boundary condition from the Serpent solution.

The Jupyter Notebook containing work completed for calculating the nodal expansion method homogenized flux distributions and discontinuity factors is provided below for reference:

:ref:`NEMnotebook`

=====================
Methodology
=====================

Results are read in from Serpent using ``serpentTools``. Using the theory described in the :ref:`Introduction`, the class ``CartesianNem1D`` is developed. When initializing the class, required arguments include ``dx``, ``dy``, ``xs``, ``bc``, ``trL``, and ``symbolic``. ``trL`` is a dictionary containing information such as interface leakage, diffusion coefficients, and surface lengths.
``symbolic`` is a Boolean which is set to ``True`` if symbolic solving is desired using ``Sympy``. If set to ``False``, the code instead completes all calculations using manually evaluated integrals and derivative functions, thereby improving efficiency.
Numerous auxiliary functions live within the ``CartesianNem1D`` class and will not be discussed in detail here. All functions are contained within ``NEM.py``.
 
=================
Results
=================

Flux profiles are generated by group and for each node within the SMR colorset initially described for the problem. The results presented are for homogeneous flux distributions calculated as a function of *y*. *x* transverse leakages are presented as a function of *y* for both fast and thermal groups for each node.
Finally, discontinuity factors for each node face (solving in the *y* direction) are presented.

---------------------------------------------------------
Homogeneous Flux Distributions
---------------------------------------------------------

The fast and thermal flux distributions are plotted for the left and right columns of the SMR colorset. The heterogeneous solution produced by Serpent is provided as a reference for comparing the NEM calculated homogenized flux. Both infinite transport cross sections and CMM transport cross sections from Serpent are used to calculate the homogenized flux profiles.

.. figure:: NEM_DF_files/NEM_DF_32_1.png
  :width: 600
  :align: center

  Fast flux distributions for SMR colorset column 1 (F2/F12).

.. figure:: NEM_DF_files/NEM_DF_33_1.png
  :width: 600
  :align: center

  Thermal flux distributions for SMR colorset column 1 (F2/F12).

.. figure:: NEM_DF_files/SMR_column2_fastflux.png
  :width: 600
  :align: center

  Fast flux distributions for SMR colorset column 2 (F11/Ref).

.. figure:: NEM_DF_files/SMR_column2_thermalflux.png
  :width: 600
  :align: center

  Thermal flux distributions for SMR colorset column 2 (F11/Ref).

  The transverse leakages are plotted separately and provided for each column of the colorset below.

.. figure:: NEM_DF_files/SMR_column1_trL.png
  :width: 600
  :align: center

  Transverse leakage as a function of *y* for SMR colorset column 1 (F2/F12).

.. figure:: NEM_DF_files/SMR_column2_trL.png
  :width: 600
  :align: center

  Transverse leakage as a function of *y* for SMR colorset column 2 (F11/Ref).

--------------------------
Discontinuity Factors
--------------------------

Discontinuity factors are presented for each north/south interface per node. Both NEM calculated and Serpent calculated values are shown.

*Column 1 Discontinuity Factors*

============== =========================== ============================
Location          Fast DF (NEM/Serpent)      Thermal DF (NEM/Serpent)
============== =========================== ============================
F2 (South)           1.017/1.026                  1.074/1.063
-------------- --------------------------- ----------------------------
F2 (North)           1.018/1.006                  1.034/1.057
-------------- --------------------------- ----------------------------
F12 (South)          0.995/1.006                  0.954/0.951
-------------- --------------------------- ----------------------------
F12 (North)          1.008/0.997                  0.974/0.973
============== =========================== ============================

*Column 2 Discontinuity Factors*

============== =========================== ============================
Location          Fast DF (NEM/Serpent)      Thermal DF (NEM/Serpent)
============== =========================== ============================
F11 (South)           1.000/0.990                  0.927/0.937
-------------- --------------------------- ----------------------------
F11 (North)           1.030/1.048                  1.084/1.027
-------------- --------------------------- ----------------------------
Ref (South)          0.978/0.974                  0.880/1.012
-------------- --------------------------- ----------------------------
Ref (North)          1.045/1.052                 2.482/1.012
============== =========================== ============================

=================
Conclusions
=================

The nodal expansion method for calculating homogenized nodal fluxes using a transverse integration method produces flux distributions representative of the heterogeneous solutions reported by Serpent. This method achieves the desired goal of implementing a quick and relatively accurate solver useful when solving the full core heterogeneous solution could be prohibitive.
Next steps will look to accomplish pin power reconstruction from homogenized 2D flux distribution.

Return to the top of the page: :ref:`proj4`
