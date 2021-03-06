
.. module:: webbpsf

=============================
The WebbPSF module
=============================


This module provides the primary interface, both for programmers and for interactive non-GUI use. It provides 
five classes corresponding to the JWST instruments, with consistent interfaces.  See below for the detailed API; for now let's dive into some example code.

:ref:`Additional code examples <more_examples>` are available later in this documentation.


Usage and Examples
-------------------

Simple PSFs are easily obtained: 

>>> import webbpsf
>>> nc = webbpsf.NIRCam()
>>> nc.filter =  'F200W'
>>> psf = nc.calcPSF(oversample=4)      # returns a pyfits.HDUlist containing PSF and header
>>> pylab.imshow(psf[0].data]           # display it on screen yourself, or
>>> display_PSF(psf)                    # use this convenient function to make a nice log plot with labeled axes
>>>
>>> psf = nc.calcPSF(filter='F470N', oversample=4)    # this is just a shortcut for setting the filter, then computing a PSF
>>>
>>> nc.calcPSF("myPSF.fits", filter='F480M' )         # you can also write the output directly to disk if you prefer.


For interactive use, you can have the PSF displayed as it is computed:

>>> nc.calcPSF(display=True)                          # will make nice plots with matplotlib.

.. image:: ./fig1_nircam_f200w.png
   :scale: 75%
   :align: center
   :alt: Sample PSF image


More complicated instrumental configurations are available by setting the instrument's attributes. For instance,
one can create an instance of MIRI and configure it for coronagraphic observations, thus:

>>> miri = webbpsf.MIRI()
>>> miri.filter = 'F1065C'
>>> miri.image_mask = 'FQPM1065'
>>> miri.pupil_mask = 'MASKFQPM'
>>> miri.calcPSF('outfile.fits')

.. image:: ./fig_miri_coron_f1065c.png
   :scale: 75%
   :align: center
   :alt: Sample PSF image


WebbPSF can output a log of calculation steps while it runs, which can be displaye to the screen and optionally saved to a file. 
This is useful for verifying or debugging calculations.  To turn on log display, just run

>>> webbpsf.setup_logging(filename='webbpsf.log')

The setup_logging function allows selection of the level of log detail following the standard Python logging system (DEBUG, INFO, WARN, ERROR). WebbPSF remembers your
chosen logging settings between invocations, so if you close and then restart python it will automatically continue logging at the same level of detail as before.
See :py:func:`setup_logging` for more details.



Input Source Spectra
^^^^^^^^^^^^^^^^^^^^^^^^^^^

WebbPSF attempts to calculate realistic weighted broadband PSFs taking into account both the source spectrum and the instrumental spectral response. 

The default source spectrum is, if :py:mod:`pysynphot` is installed, a G2V star spectrum from Castelli & Kurucz 2004. Without :py:mod:`pysynphot`, the default is a simple flat spectrum such that the same number of photons are detected at each wavelength.

You may choose a different illuminating source spectrum by specifying a ``source`` parameter in the call to ``calcPSF()``. The following are valid sources:

1. A :py:class:`pysynphot.Spectrum` object. This is the best option, providing maximum ease and accuracy, but requires the user to have :py:mod:`pysynphot` installed.  In this case, the :py:class:`Spectrum` object is combined with a :py:class:`pysynphot.ObsBandpass` for the selected instrument and filter to derive the effective stimulus in detected photoelectrons versus wavelength. This is binned to the number of wavelengths set by the ``nlambda`` parameter. 
2. A dictionary with elements ``source["wavelengths"]`` and ``source["weights"]`` giving the wavelengths in meters and the relative weights for each. These should be numpy arrays or lists. In this case, the wavelengths and weights are used exactly as provided, without applying the instrumental filter profile. 

   >>> src = {'wavelengths': [2.0e-6, 2.1e-6, 2.2e-6], 'weights': [0.3, 0.5, 0.2]}
   >>> nc.calcPSF(source=src, outfile='psf_for_src.fits')

3. A tuple or list containing the numpy arrays ``(wavelength, weights)`` instead.


To calculate a monochromatic PSF, just use the ``monochromatic`` parameter. Wavelengths are always specified in meters. This is just a shorthand for a single-element ``source`` dict.

   >>> miri.calcPSF(monochromatic=9.876e-6)


As a convenience, webbpsf includes a function to retrieve an appropriate :py:class:`pysynphot.Spectrum` object for a given stellar spectral type from the PHOENIX or Castelli & Kurucz model libraries. 

   >>> src = webbpsf.specFromSpectralType('G0V', catalog='phoenix')
   >>> psf = miri.calcPSF(source=src)

Input Source position offsets
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The PSF may be shifted off-center by adjusting the offset of the stellar source. This is done in polar coordinates:

>>> instrument.options['source_offset_r'] = 0.3         # offset in arcseconds
>>> instrument.options['source_offset_theta'] = 45.     # degrees counterclockwise from instrumental +Y in the science frame

If these options are set, the offset is applied relative to the central coordinates as defined by the output array size and parity (described just below).

For coronagraphic modes, the coronagraph occulter is always assumed to be at the center of the output array. Therefore, these options let you offset the source away from the coronagraph.



Array sizes, star positions, and centering
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Output array sizes may be specified either in units of arcseconds or pixels.  For instance, 

>>> mynircam = webbpsf.NIRCam()
>>> result = mynircam.calcPSF(fov_arcsec=7, oversample=2, filter='F250M')
>>> result2= mynircam.calcPSF(fov_pixels=512, oversample=2, filter='F250M')

In the latter example, you will in fact get an array which is 1024 pixels on a side: 512 physical detector pixels, times an oversampling of 2.


By default, the PSF will be centered at the exact center of the output array. This means that if the PSF is computed on an array with an odd number of pixels, the
PSF will be centered exactly on the central pixel. If the PSF is computed on an array with even size, it will be centered on the "crosshairs" at the intersection of the central four pixels.
If one of these is particularly desirable to you, set the parity option appropriately:

>>>  instrument.options['parity'] = 'even'
>>>  instrument.options['parity'] = 'odd'

Setting one of these options will ensure that a field of view specified in arcseconds is properly rounded to either odd or even when converted from arcsec to pixels. Alternatively, 
you may also just set the desired number of pixels explicitly in the call to calcPSF():

>>>  instrument.calcPSF(fov_npixels = 512)


.. note::

    Please note that these parity options apply to the number of *detector
    pixels* in your simulation. If you request oversampling, then the number of
    pixels in the output file for an oversampled array will be
    ``fov_npixels`` times ``oversampling``. Hence, if you request an odd
    parity with an even oversampling of, say, 4, then you would get an array
    with a total number of data pixels that is even, but that correctly represents
    the PSF located at the center of an odd number of detector pixels.



Pixel scales, sampling, and oversampling
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The derived instrument classes all know their own instrumental pixel scales. You can change the output 
pixel scale in a variety of ways, as follows. See the :py:class:`JWInstrument.calcPSF` documentation for more details.

1. Set the ``oversample`` parameter to calcPSF(). This will produce a PSF with a pixel grid this many times more finely sampled. 
   ``oversample=1`` is the native detector scale, ``oversample=2`` means divide each pixel into 2x2 finer pixels, and so forth.
   You can automatically obtain both the oversampled PSF and a version rebinned down onto the detector pixel scale by setting `rebin=True` 
   in the call to calcPSF:

   >>> hdulist = instrument.calcPSF(oversample=2, rebin=True)     # hdulist will contain a primary HDU with the 
   >>>                                                            # oversampled data, plus an image extension 
   >>>                                                            # with the PSF rebinned down to regular sampling.

   

2. For coronagraphic calculations, it is possible to set different oversampling factors at different parts of the calculation. See the ``calc_oversample`` and ``detector_oversample`` parameters. This
   is of no use for regular imaging calculations (in which case ``oversample`` is a synonym for ``detector_oversample``). Specifically, the ``calc_oversample`` keyword is used for Fourier transformation to and from the intermediate optical plane where the occulter (coronagraph spot) is located, while ``detector_oversample`` is used for propagation to the final detector. Note that the behavior of these keywords changes for coronagraphic modeling using the Semi-Analytic Coronagraphic propagation algorithm (not fully documented yet - contact Marshall Perrin if curious). 

   >>> miri.calcPSF(calc_oversample=8, detector_oversample= 2)    # model the occulter with very fine pixels, then save the 
   >>>                                                           # data on a coarser (but still oversampled) scale

3. Or, if you need even more flexibility, just change the ``instrument.pixelscale`` attribute to be whatever arbitrary scale you require. 

   >>> instrument.pixelscale = 0.0314159


 
Note that the calculations performed by WebbPSF are somewhat memory intensive, particularly for coronagraphic observations. All arrays used internally are 
double-precision complex floats (16 bytes per value), and many arrays of size (npixels*oversampling)^2 are needed (particularly if display options are turned on, since the
Matplotlib graphics library makes its own copy of all arrays displayed). Your average laptop with a couple GB of RAM will do perfectly well for most computations so long as you're not too ambitious with setting array size and oversampling. 
If you're interested in very high fidelity simulations of large fields (e.g. 1024x1024 pixels oversampled 8x) then we recommend a large multicore desktop with >16 GB RAM. 



Output format options for sampling
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As just explained, WebbPSF can easily calculate PSFs on a finer grid than the detector's native pixel scale. You can select whether the output data should include this oversampled image, a copy that has instead been rebinned down to match the detector scale, or optionally both. This is done using the ``options['output_mode']`` parameter. 

   >>> nircam.options['output_mode'] = 'oversampled'      
   >>> psf = nircam.calcPSF()       # the 'psf' variable will be an oversampled PSF, formatted as a FITS HDUlist
   >>>
   >>> nircam.options['output_mode'] = 'detector sampled'      
   >>> psf2 = nircam.calcPSF()      # now 'psf2' will contain the result as resampled onto the detector scale.
   >>>
   >>> nircam.options['output_mode'] = 'both'      
   >>> psf3 = nircam.calcPSF()      # 'psf3' will have the oversampled image as primary HDU, and 
   >>>                              # the detector-sampled image as the first image extension HDU.


The default behavior is 'both'. Note that at some point in the future, this default is likely to change to detector sampling. 


Detailed WebbPSF API Reference
-------------------------------

Class hierarchy
^^^^^^^^^^^^^^^^


.. inheritance-diagram:: webbpsf.NIRCam webbpsf.NIRSpec webbpsf.MIRI webbpsf.NIRISS webbpsf.FGS



The JWInstrument generic class
#################################

.. autoclass:: webbpsf.JWInstrument
   :members:


.. _specific_instrument:

Notes on Specific Instruments
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
NIRCam
#######

.. autoclass:: webbpsf.NIRCam

        See methods under :py:class:`JWInstrument` 
        

NIRSpec
###########

.. autoclass:: webbpsf.NIRSpec

        See methods under :py:class:`JWInstrument` 

MIRI
######

.. autoclass:: webbpsf.MIRI

        See methods under :py:class:`JWInstrument` 

.. figure:: ./fig_miri_f1000w.png
   :scale: 75%
   :align: center
   :alt: Sample PSF image for MIRI

   An example MIRI PSF in F1000W. 

   Note that the MIRI imager field of view is rotated by 4.56 degrees relative to the JWST pupil; the coronagraph optics are
   correspondingly counterrotated to align them with the pupil.  For direct imaging PSF calculations, this is most simply handled by
   rotating the pupil mask and OPD file prior to the Fourier propagation. For MIRI coronagraphy on the other hand, the rotation is performed as the 
   last step prior to the detector. 
   
   Technical aside: Note that for computational reasons having to do with accurately simulating PSF centering on an FQPM, MIRI coronagraphic
   simulations will include two 'virtual optics' called 'FQPM FFT aligners' that  will show up in the display window for such calculations. These 
   can be ignored by most end users of this software; interested readers should consult the  :py:mod:`POPPY <poppy>` documentation for more detail.

NIRISS
#######

.. autoclass:: webbpsf.NIRISS

        See methods under :py:class:`JWInstrument` 




FGS
#####

.. autoclass:: webbpsf.FGS

        See methods under :py:class:`JWInstrument` 


TFI
#####

Deprecated in favor of NIRISS.  

Utility Functions for Display and Plotting
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


.. autofunction:: webbpsf.setup_logging

.. autofunction:: webbpsf.Instrument

Display Functions
#####################

.. autofunction:: display_PSF
.. autofunction:: display_PSF_difference
.. autofunction:: display_EE
.. autofunction:: display_profiles

Metrics of PSF Quality
#########################

.. autofunction:: radial_profile
.. autofunction:: measure_EE
.. autofunction:: measure_fwhm
.. autofunction:: measure_sharpness
.. autofunction:: measure_centroid

--------------

Documentation last updated on |today|


