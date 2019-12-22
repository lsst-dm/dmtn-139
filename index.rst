


..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This document describes the architecture for Internet-visible image services to be provided by LSST.  It explains the use of IVOA standards to provide these services, and discusses where LSST's needs may require extensions to those standards.  It addresses the back-end architecture that supports these services, including especially the role of the Gen3 Butler.

Image services include: observation metadata services based on ObsCore and CAOM2, image retrieval services, image cutout and mosiacing services, image-recreation-on-demand, and HiPS multi-resolution image map services.

This note discusses the role of the image services in the LSST Science Platform and outlines expectations for how they will be used internally and externally.

Later editions of this note may discuss issues associated with IVOA Registry representation of the services, or they may be moved to a separate technote on our Registry interface.


Introduction
============

LSST, during commissioning and operations, will generate a wide variety of image data products.
These will include both single-epoch data, including raw, calibrated, and difference images, and coadded data of various types, including multi-resolution all-sky maps.
LSST will provide access to both precomputed images and created-on-demand images.
On-demand image creation may have latencies of up to hours in duration.

Under the "VO First" strategy, we are committed to providing, as part of the API Aspect of the LSST Science Platform, image access through a basic set of IVOA-compliant services, as well as a richer interface that provides additional metadata.
We expect to use these interfaces from the other Aspects of the Science Platform.

We refer to all these interfaces generically as "image services"; however, they are in fact a combination of image *metadata* services and services for actual image pixel data.
The distinction will be made clear whenever possible.

Users will be interested in the relationships between the single-epoch images and the coadded images, and between single-epoch images of different types, and our architecture should facilitate the discovery and exploration of these relationships.

From a design standpoint, the image service architecture must set out the relationship between the "Butler" (Gen3) view of access to image data and the external services provided.

This document, when complete, will review the established LSST requirements applicable to image services, document a set of reference use cases that we plan to satisfy, describe the overall concept for image services


Requirements
============

SRD (LPM-17)
------------

There are few substantive requirements at SRD level on image access *per se*, but there are some remarks on images and image metadata which are worth restating.
In Section 3.5, "Data Processing and Management Requirements", it is stated:

- "Level 2 data products will be made available as annual Data Releases and will include images [...]" - this simply states that images are part of the DRs.
- "Observing conditions, such as point-spread function and background level, will be evaluated and known for each visit" - this suggests the availability of this sort of information as a metadata catalog.
- "Images and catalog data will be released to publicly-accessible repositories as soon as possible after they are obtained." - this implies the existence of an accessible image data service for the Prompt (Level 1) image data products.
- "fixed 'snapshots' of the data should periodically be released [including] images, corrected for instrumental artifacts, and photometrically and astrometrically calibrated, will be released to [the] public every DRT1 [1.0] years [...].  [...] images will be released for both single visits and for appropriately co-added data (such as those optimized for depth and for weak lensing analysis)." - this clarifies the availability of both broard types of image data products.



LSR (LSE-29)
------------

- LSR-REQ-0015, Data Format: "The LSST survey data shall be collected in the form of pixel addressable digital images that preserve the full information content of the LSST instrument."
- LSR-REQ-0019, Ancillary Data: "The LSST system shall measure and record the data required as input to the optimization of the acquisition of survey data as well as record the environmental conditions that existed during each exposure. These data shall include but are not limited to 1) Atmospheric seeing; 2) Cloud cover; and 3) Meteorological information (temperature, wind, humidity, etc.)."
- LSR-REQ-0021, Calibrated Image Production: "The LSST data processing system shall process raw image data to produce photometrically and astrometrically calibrated images, both from single visits and from deep coadds."
- LSR-REQ-0110, Level 1 Scientific Content: "The Level 1 data products shall include: 1) Raw Science Images; 2) Calibrated Science Images (trimmed, de-biased, flattened, etc.), 3) Difference Images, 4) Image Metadata/Catalog, ..."
- LSR-REQ-0104, Level 1 Data Product Availability: "All Level 1 Data Products except Transient Alerts & Solar System Objects shall be produced and made available to the consortium not later than L1PublicT of the acquisition of the corresponding raw images."
- LSR-REQ-0126, Level 1 Data Product Embargo: "LSST shall not release image or catalog data resulting from a visit, except for the content of the public alert stream, sooner than time L1PublicTMin [6 hours] following the acquisition of the raw image data from that visit."
- LSR-REQ-0105, Science Data Quality Archiving: "SDQA data produced shall be archived in association with the corresponding raw image data."
- LSR-REQ-0037, Level 2 Scientific Content: "The Level 2 Data Products in a Data Release shall include: 1) Images, corrected for instrumental artifacts and photometrically and astrometrically calibrated, [...], 7) Deep co-added images of the full survey area on the sky."
- LSR-REQ-0053, Data Distribution: "The LSST shall permit and facilitate the bulk distribution of its public data to remote sites or users wishing to consume or host it, subject to the availability of resources and the data access policy from LSR-REQ-0052."
- LSR-REQ-0054, Data Product Access Interface: "The LSST shall provide access to all its public data products through an interface that utilizes, to the maximum practicable extent, community-based standards such as those for pixel-based images (e.g., FITS), as well as those being developed by the Virtual Observatory (VO) community, and that facilitates user data analysis and the production of Level 3 and other user-defined data products at LSST-provided facilities and at remote sites."



OSS (LSE-30)
------------

DMSR (LSE-61)
-------------

LSP Requirements (LDM-554)
--------------------------


Use Cases
=========

Service Concept
===============

Image Data Formats
------------------

Observation Metadata
--------------------

CAOM2
^^^^^

ObsCore
^^^^^^^

Special Case: HiPS
------------------

Service Details
===============

ObsTAP Service
--------------

TAP Service for Observation Metadata
------------------------------------

SIAv2 Service
-------------

SODA Service(s)
---------------

HiPS Service
------------

User Access Scenarios
=====================

Portal Aspect
-------------

Minimal (Frozen) Portal
^^^^^^^^^^^^^^^^^^^^^^^

Aspirational Portal Design
^^^^^^^^^^^^^^^^^^^^^^^^^^

Notebook Aspect and External Access
-----------------------------------

PyVO & Astroquery

Development of PyVO SIAv2 capability.



.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
