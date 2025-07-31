:orphan:

.. _LatticeParam:

Lattice Steps and NEM Solutions
===============================

Return to :ref:`proj6` documentation.

The cross sections for the non-reflector fuel assemblies and the corner
DFs are obtained from the infinite lattice calculations. These values
are used in subsequent NEM and AFEN method calculations.

.. code:: python

    # import relevant packages
    from IPython.display import Image
    import numpy as np
    import matplotlib.pyplot as plt
    from matplotlib import rcParams
    import serpentTools
    from serpentTools.settings import rc
    from analytic_nodal_expansion import AFEN2D, meshPlot, GetSerpentRes
    from NEM import CartesianNem1D, Plot1d
    FONT_SIZE = 16
    plt.rcParams['figure.figsize'] = [6, 4] # Set default figure size

.. code:: python

    # Read in Serpent data for each unique fuel assembly type
    detFile455 = './serpent/Fuel_only/Fuel_455_2g_900K_det0.m'
    resFile455 = './serpent/Fuel_only/Fuel_455_2g_900K_res.m'
    detFile260 = './serpent/Fuel_only/Fuel_260_2g_det0.m'
    resFile260 = './serpent/Fuel_only/Fuel_260_2g_res.m'
    
    rc["serpentVersion"] = "2.2.1"
    det455 = serpentTools.read(detFile455)
    res455 = serpentTools.read(resFile455)
    det260 = serpentTools.read(detFile260)
    res260 = serpentTools.read(resFile260)

4.55% Enriched Fuel Assembly Data and 2.60% Enriched Fuel Assembly Data
using GetSerpentRes

.. code:: python

    xs455, bc455 = GetSerpentRes(resFile455, 'F1', 0)
    xs260, bc260 = GetSerpentRes(resFile260, 'F1', 0)
    
    # XS and BC stored as dictionaries
    bc = {'F2': bc455, 'F11': bc260}
    xs = {'F2': xs455, 'F11': xs260}
    
    # Calculating Corner Discontinuity Factors from infinite lattice results for each assembly (eq. 13.6)
    cdf455 = bc['F2']['neFlux']/bc['F2']['flux']
    cdf260 = bc['F11']['neFlux']/bc['F11']['flux']

Obtain Group-wise Form Functions

.. code:: python

    npins = 17
    dx, dy = 21.42, 21.42  # cm  - assembly length
    fastPower455 = det455.detectors['power_fast'].tallies[0:npins, 0:npins]
    thermalPower455 = det455.detectors['power_thermal'].tallies[0:npins, 0:npins]
    fastPower260 = det260.detectors['power_fast'].tallies[0:npins, 0:npins]
    thermalPower260 = det260.detectors['power_thermal'].tallies[0:npins, 0:npins]
    
    # 4.55% form factors
    ff455 = {}
    ff455[0] = fastPower455/np.average(fastPower455); ff455[1] = thermalPower455/np.average(thermalPower455)
    
    # 2.60% form factors
    ff260 = {}
    ff260[0] = fastPower260/np.average(fastPower260); ff260[1] = thermalPower260/np.average(thermalPower260)

Generate mesh plots for form factors

.. code:: python

    # plt.figure()
    # meshPlot(ff455[0], npins, 'F2 (Fast)', 'ff')
    # meshPlot(ff455[1], npins, 'F2 (Thermal)', 'ff')
    # meshPlot(ff260[0], npins, 'F11/F12 (Fast)', 'ff')
    # meshPlot(ff260[1], npins, 'F11/F12 (Thermal)', 'ff')

Begin Nodal Expansion Method (NEM) solution for full colorset

.. code:: python

    detFile = './serpent/SMR/SMR_Ref_2D_2g_det0.m'
    resFile = './serpent/SMR/SMR_Ref_2D_2g_res.m'
    
    det = serpentTools.read(detFile)

Solutions in the x-direction for row 1 (F2/F11) and row 2 (F12/Ref)

.. code:: python

    xtally = det.detectors['flux_fast'].x[:, 1]
    # Row 1 Data
    fastHetFluxX1 = det.detectors['flux_fast'].tallies[0:npins].mean(axis=0)
    thermalHetFluxX1 = det.detectors['flux_thermal'].tallies[0:npins].mean(axis=0)
    universesX1 = ['F2', 'F11']
    # Row 2 Data
    fastHetFluxX2 = det.detectors['flux_fast'].tallies[npins:].mean(axis=0)
    thermalHetFluxX2 = det.detectors['flux_thermal'].tallies[npins:].mean(axis=0)
    universesX2 = ['F12', 'Ref']
    
    xsX11, bcX11 = GetSerpentRes(resFile, universesX1[0], timeDays=0)
    xsX12, bcX12 = GetSerpentRes(resFile, universesX1[1], timeDays=0)
    
    # Define transverse leakage terms
    # F2
    trLeakageX11 = {}
    trLeakageX11['eL'] = bcX12['nJnet'] - bcX12['sJnet']
    trLeakageX11['eD'] = xsX12['diff']
    trLeakageX11['edx'] = dx
    trLeakageX11['wL'] = bcX11['nJnet'] - bcX11['sJnet']
    trLeakageX11['wD'] = xsX11['diff']
    trLeakageX11['wdx'] = dx
    # F11
    trLeakageX12 = {}
    trLeakageX12['wL'] = bcX11['nJnet'] - bcX11['sJnet']
    trLeakageX12['wD'] = xsX11['diff']
    trLeakageX12['wdx'] = dx
    trLeakageX12['eL'] = bcX12['nJnet'] - bcX12['sJnet']
    trLeakageX12['eD'] = xsX12['diff']
    trLeakageX12['edx'] = dx
    
    xvals = np.linspace(-dx/2, +dx/2, npins)
    # F2
    nemX11 = CartesianNem1D(dx, dy, xsX11, bcX11, trLeakageX11, symbolic=False)
    nemX11.TransverseLeakageCoef('x')
    nemX11.GetExpansionCoeffs('x', 'diff')
    fluxX11 = nemX11.GetHomogFlux([-dx/2, dx/2])
    # F11
    nemX12 = CartesianNem1D(dx, dy, xsX12, bcX12, trLeakageX12, symbolic=False)
    nemX12.TransverseLeakageCoef('x')
    nemX12.GetExpansionCoeffs('x', 'diff')
    fluxX12 = nemX12.GetHomogFlux([-dx/2, dx/2])
    
    # Solution for row 2 data
    xsX21, bcX21 = GetSerpentRes(resFile, universesX2[0], timeDays=0)
    xsX22, bcX22 = GetSerpentRes(resFile, universesX2[1], timeDays=0)
    
    # Define transverse leakage terms
    # F12
    trLeakageX21 = {}
    trLeakageX21['eL'] = bcX22['nJnet'] - bcX22['sJnet']
    trLeakageX21['eD'] = xsX22['diff']
    trLeakageX21['edx'] = dx
    trLeakageX21['wL'] = bcX21['nJnet'] - bcX21['sJnet']
    trLeakageX21['wD'] = xsX21['diff']
    trLeakageX21['wdx'] = dx
    # Ref
    trLeakageX22 = {}
    trLeakageX22['wL'] = bcX21['nJnet'] - bcX21['sJnet']
    trLeakageX22['wD'] = xsX21['diff']
    trLeakageX22['wdx'] = dx
    trLeakageX22['eL'] = bcX22['nJnet'] - bcX22['sJnet']
    trLeakageX22['eD'] = xsX22['diff']
    trLeakageX22['edx'] = dx
    
    # F12
    nemX21 = CartesianNem1D(dx, dy, xsX21, bcX21, trLeakageX21, symbolic=False)
    nemX21.TransverseLeakageCoef('x')
    nemX21.GetExpansionCoeffs('x', 'diff')
    fluxX21 = nemX21.GetHomogFlux([-dx/2, dx/2])
    # Ref
    nemX22 = CartesianNem1D(dx, dy, xsX22, bcX22, trLeakageX22, symbolic=False)
    nemX22.TransverseLeakageCoef('x')
    nemX22.GetExpansionCoeffs('x', 'diff')
    fluxX22 = nemX22.GetHomogFlux([-dx/2, dx/2])

Solutions in the y-direction for column 1 (F2/F12) and column 2
(F11/Ref)

.. code:: python

    ytally = xtally
    # Column 1 data
    fastHetFluxY1 = det.detectors['flux_fast'].tallies[:,0:npins].mean(axis=1)
    thermalHetFluxY1 = det.detectors['flux_thermal'].tallies[:,0:npins].mean(axis=1)
    universesY1 = ['F2', 'F12']
    # Column 2 data
    fastHetFluxY2 = det.detectors['flux_fast'].tallies[:,npins:].mean(axis=1)
    thermalHetFluxY2 = det.detectors['flux_thermal'].tallies[:,npins:].mean(axis=1)
    universesY2 = ['F11', 'Ref']
    
    xsY11, bcY11 = GetSerpentRes(resFile, universesY1[0], timeDays=0)
    xsY21, bcY21 = GetSerpentRes(resFile, universesY1[1], timeDays=0)
    
    # Define transverse leakage terms
    # F2
    trLeakageY11 = {}
    trLeakageY11['nL'] = bcY21['eJnet'] - bcY21['wJnet']
    trLeakageY11['nD'] = xsY21['diff']
    trLeakageY11['ndy'] = dy
    trLeakageY11['sL'] = bcY11['eJnet'] - bcY11['wJnet']
    trLeakageY11['sD'] = xsY11['diff']
    trLeakageY11['sdy'] = dy
    # F12
    trLeakageY21 = {}
    trLeakageY21['sL'] = bcY11['eJnet'] - bcY11['wJnet']
    trLeakageY21['sD'] = xsY11['diff']
    trLeakageY21['sdy'] = dy
    trLeakageY21['nL'] = bcY21['eJnet'] - bcY21['wJnet']
    trLeakageY21['nD'] = xsY21['diff']
    trLeakageY21['ndy'] = dy
    
    yvals = np.linspace(-dy/2, +dy/2, npins)
    # F2
    nemY11 = CartesianNem1D(dx, dy, xsY11, bcY11, trLeakageY11, symbolic=False)
    nemY11.TransverseLeakageCoef('y')
    nemY11.GetExpansionCoeffs('y', 'diff')
    fluxY11 = nemY11.GetHomogFlux([-dy/2, dy/2])
    # F12
    nemY21 = CartesianNem1D(dx, dy, xsY21, bcY21, trLeakageY21, symbolic=False)
    nemY21.TransverseLeakageCoef('y')
    nemY21.GetExpansionCoeffs('y', 'diff')
    fluxY21 = nemY21.GetHomogFlux([-dy/2, dy/2])
    
    # Solution for column 2 data
    xsY12, bcY12 = GetSerpentRes(resFile, universesY2[0], timeDays=0)
    xsY22, bcY22 = GetSerpentRes(resFile, universesY2[1], timeDays=0)
    
    # F11
    trLeakageY12 = {}
    trLeakageY12['nL'] = bcY22['eJnet'] - bcY22['wJnet']
    trLeakageY12['nD'] = xsY22['diff']
    trLeakageY12['ndy'] = dy
    trLeakageY12['sL'] = bcY12['eJnet'] - bcY12['wJnet']
    trLeakageY12['sD'] = xsY12['diff']
    trLeakageY12['sdy'] = dy
    
    # Ref
    trLeakageY22 = {}
    trLeakageY22['nL'] = bcY22['eJnet'] - bcY22['wJnet']
    trLeakageY22['nD'] = xsY22['diff']
    trLeakageY22['ndy'] = dy
    trLeakageY22['sL'] = bcY12['eJnet'] - bcY12['wJnet']
    trLeakageY22['sD'] = xsY12['diff']
    trLeakageY22['sdy'] = dy
    
    # F11
    nemY12 = CartesianNem1D(dx, dy, xsY12, bcY12, trLeakageY12, symbolic=False)
    nemY12.TransverseLeakageCoef('y')
    nemY12.GetExpansionCoeffs('y', 'diff')
    fluxY12 = nemY12.GetHomogFlux([-dy/2, dy/2])
    # Ref
    nemY22 = CartesianNem1D(dx, dy, xsY22, bcY22, trLeakageY22, symbolic=False)
    nemY22.TransverseLeakageCoef('y')
    nemY22.GetExpansionCoeffs('y', 'diff')
    fluxY22 = nemY22.GetHomogFlux([-dy/2, dy/2])

Obtain the homogeneous surface flux at each face

.. code:: python

    # stored as dictionary in format used in AFEN calculation
    surffluxhom = {
        "F2": {
            "w": fluxX11[:, 0],
            "s": fluxY11[:, 0],
            "e": fluxX11[:, 1],
            "n": fluxY11[:, 1],
        },
        "F11": {
            "w": fluxX12[:, 0],
            "s": fluxY12[:, 0],
            "e": fluxX12[:, 1],
            "n": fluxY12[:, 1],
    
        },
        "F12": {
            "w": fluxX21[:, 0],
            "s": fluxY21[:, 0],
            "e": fluxX21[:, 1],
            "n": fluxY21[:, 1],
        },
        "Ref": {
            "w": fluxX22[:, 0],
            "s": fluxY22[:, 0],
            "e": fluxX22[:, 1],
            "n": fluxY22[:, 1], 
        }
    }

.. code:: python

    # plt.figure()
    # Plot1d(xvals-10.71, fluxY12[0,:], xlabel="position, cm", 
    #        ylabel='Flux distribution',
    #        fontsize=16, marker="-r", markersize=6)
    # Plot1d(xvals+10.71, fluxY22[0,:], xlabel="position, cm", 
    #        ylabel='Flux distribution',
    #        fontsize=16, marker="-b", markersize=6)
    # Plot1d(xtally, fastHetFluxY2, xlabel="position, cm", 
    #        ylabel='Fast flux distribution',
    #        fontsize=16, marker="-k", markersize=6)
    # plt.show()
