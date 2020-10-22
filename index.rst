


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

:tocdepth: 2

.. Please do not modify tocdepth to a value other than 1; will be fixed when a new Sphinx theme is shipped.

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

The relevant requirements from the LSST document tree are excerpted below.
They have been lightly edited for compactness (such as by compressing bullet points to in-line lists), but the language is intact.
In some cases, excerpts from the "Discussion" sections of requirements have been included if they appear particular useful in clarifying the requirement for the purposes of the present document.
At this time the requirements have largely been left in the order they appear in their source documents, rather than attempting to regroup them topically.

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
- LSR-REQ-0048, Raw Image Data Archiving: "The LSST system shall archive all raw science and calibration image data, collected in the course of the survey as well as data collected during engineering and calibration operations, as well as all wavefront sensor data.  It shall also permit the archiving of such diagnostic image data as may be needed to support the commissioning, calibration, and maintenance of the observatory."
- LSR-REQ-0108, Meta Data Archiving: "The LSST system shall archive sufficient information to permit the reliable and reproducible retrieval of calibrated image data.  *Discussion:* Calibrated image data must be available to retrieve, but may be reconstructed on demand as an alternative to its direct archiving (see LSR-REQ-0049)."
- LSR-REQ-0049, Data Product Archiving: "The LSST system shall archive all generated Level 1, Level 2, and Calibration Data Products, or provide services to reconstruct any given data product on demand. When regenerated on-demand, the Data Products shall be scientifically equivalent – i.e. at a level of precision sufficient to reproduce the primary and derived attributes well within their formal uncertainties."
- LSR-REQ-0053, Data Distribution: "The LSST shall permit and facilitate the bulk distribution of its public data to remote sites or users wishing to consume or host it, subject to the availability of resources and the data access policy from LSR-REQ-0052."
- LSR-REQ-0054, Data Product Access Interface: "The LSST shall provide access to all its public data products through an interface that utilizes, to the maximum practicable extent, community-based standards such as those for pixel-based images (e.g., FITS), as well as those being developed by the Virtual Observatory (VO) community, and that facilitates user data analysis and the production of Level 3 and other user-defined data products at LSST-provided facilities and at remote sites."


OSS (LSE-30)
------------

- OSS-REQ-0129, Exposures (Level 1): "The Level 1 Data Products shall include the following types of Exposures: 1) Raw Exposures as obtained from the Camera and assembled/re-formatted for processing; 2) Processed Exposures (trimmed, de-biased, flattened, etc. - i.e., provisionally calibrated); 3) Difference Exposures.  Discussion: All exposures, other than raw exposures, may be regenerated on demand from raw data, rather than be physically stored." - NB: the choice of regenerating on-demand vs. storage was to be left as an implementation-era, or even operations-era, choice to enable performing appropriate trades based on then-year costs.
- OSS-REQ-0130, Catalogs (Level 1): "The Level 1 Data Products shall include the following catalogs: 1) Exposure meta-data, [...]"
- OSS-REQ-0133, Level 2 Data Products: "Level 2 Data Products shall periodically be derived from a processing pass over the complete data set."
- OSS-REQ-0134, Level 2 Data Product Availability: "All Level 2 Data Products shall be made public as part of a Data Release, with releases coming at least once every time DRT1, and as soon as possible following the completion of data release processing as is consistent with verifying the applicable data quality requirements."
- OSS-REQ-0135, Uniformly calibrated and processed versions of Level 1 Data Products: "In association with the production of the Level 2 Data Products, a data release shall also include uniformly processed and calibrated versions of all the Level 1 Data Products."
- OSS-REQ-0136, Co-added Exposures: "The Level 2 Data Products shall include the following types of co-added Exposures, covering the full exposed area of the survey: 1) Template co-adds for creating Difference Exposures, per filter band; 2) Detection co-adds for object detection, optimized for the faintest limiting magnitude; may be both multi-band and single-band; 3) Multi-band (RGB) co-adds for visualization and EPO."
- OSS-REQ-0140, Production [Level 3 Data Products]: "It shall be possible to create Level 3 Data Products either using external or internal (Data Access Center) resources, provided they meet certain requirements. LSST shall provide a set of specifications and a software toolkit to facilitate this. Level 3 Data Products may consist of new catalogs, additional data to be federated with existing catalogs, or **image data** [our emphasis]."
- OSS-REQ-0169, Data Products: "The LSST project shall archive each of its Level 1 and Level 2 data products, or the complete set of inputs and provenance necessary to reproduce it, for the lifetime of the survey. *Discussion:* Some data products are archived, while others, such as calibrated exposures, are intended to be recreated on demand."
- OSS-REQ-0177, Data Access Environment: "The LSST shall provide an open-source environment for access to its public data products, including images and catalogs."
- OSS-REQ-0178, Data Distribution: "The LSST project shall facilitate the distribution of its data products in bulk to other sites and institutions willing to host it, in accordance with LSSTC Board approved policies for data release."
- OSS-REQ-0181, Data Products Query and Download Infrastructure: "The LSST project shall provide processing, storage, and network resources for general community query and download access to LSST data products."
- OSS-REQ-0186, Access to Previous Data Releases: "The LSST Project shall provide data access services for the current Level 1 data, the most recent nDRMin [2] Data Releases, and multiple older Data Releases."
- OSS-REQ-0396, Data Access Services: "The data access services shall be designed to permit, and their software implementation shall support, the service of at least nDRTot [11] Data Releases accumulated over the (find the actual survey-length parameter) surveyYears-year [10-year] planned survey."
- OSS-REQ-0400, Subsets Support: "The data access services shall be designed to support the service of operations-designated subsets of the content of the “older Data Releases” referred to in requirement OSS-REQ-0186 from high-latency media." - In the context of this document the service of images from "high-latency media" (e.g., tape) are of particular interest.
- OSS-REQ-0262, Science Image Delivery: "The imaging system shall deliver the science data with a unique identifier per device per exposure."

DMSR (LSE-61)
-------------

Requirements on Services
^^^^^^^^^^^^^^^^^^^^^^^^

- DMS-REQ-0380 (Priority: 1b), HiPS Service: "The Data Management system shall include a secure and authenticated Internet endpoint for an IVOA-compliant HiPS service. This service shall be advertised via Registry as well as in the HiPS community mechanism operated by CDS, or whatever equivalent mechanism may exist in the LSST operations era."
- DMS-REQ-0381 (Priority: 2), HiPS Linkage to Coadds: "The HiPS maps produced by the Data Management system shall provide for straightforward linkage from the HiPS data to the underlying LSST coadded images. This SHOULD be implemented using a mechanism supported by both the LSST Science Platform and by community tools."
- DMS-REQ-0384 (Priority: 1b), Export MOCs As FITS: "The Data Management system shall provide a means for exporting the LSST-generated MOCs in the FITS serialization form defined in the IVOA MOC Recommendation."
- DMS-REQ-0340 (Priority: 2), Access Controls of Level 3 Data Products: "All Level 3 data products shall be configured to have the ability to have access restricted to the owner, a list of people, a named group, or be completely public." - Note that Level 3 data products may include user-generated images.
- DMS-REQ-0160 (Priority: 1b), Provide User Interface Services: "The DMS shall provide software for User Interface Services, including services to: browse LSST data products through astronomical views or visualizations; create and serve ”best” images of selectable regions of the sky; resample and re-project images, and visualize catalog content." - We assume that, in line with the Science Platform Vision, LSE-319, these services must include programmatic Web APIs.
- DMS-REQ-0298 (Priority: 1a), Data Product and Raw Data Access: "The DMS shall provide software for Data Access Services to list and retrieve image, file, and catalog data products (including raw telescope images and calibration data), their associated metadata, their provenance, or any combination thereof, independent of their actual storage location."
- DMS-REQ-0065 (Priority: 1b), Provide Image Access Services: "The DMS shall provide a service for Image Access through community data access protocols, to support programmatic search and retrieval of images or image cut-outs. The service shall support one or more community standard formats, including the LSST pipeline input format.  *Discussion:*  At least the FITS image format will be supported though an IVOA-standard service such as SIAP. Other image formats such as JPG may be more compatible with education/public outreach needs."
- DMS-REQ-0387 (Priority: 1b), Serve Archived Provenance: "The Data Management System shall make the archived provenance data available to science users together with the associated science data products."
- DMS-REQ-0311 (Priority: 1b), Regenerate Un-archived Data Products: "The DMS shall be able to regenerate unarchived data products to within scientifically reasonable tolerances. *Discussion:* Unarchived data products currently include Processed Visit Images for single visits, some Coadds, and Difference Images. Scientifically reasonable tolerances means well within the formal uncertainties of the data product, given the same production software, calibrations, and compute platform, all of which are expected to change (and improve) during the course of the survey."
- DMS-REQ-0336 (Priority: 1b), Regenerating Data Products from Previous Data Releases: "The DMS shall be able to regenerate data products from previous data releases to within scientifically reasonable tolerances."
- DMS-REQ-0127 (Priority: 2), Access to input images for DAC-based Level 3 processing: "The DMS shall provide access to all Level 1 and Level 2 image products through the LSST project’s Data Access Centers, and any others that have been established and funded, for Level 3 processing that takes place at the DACs."


Performance Requirements
^^^^^^^^^^^^^^^^^^^^^^^^

- DMS-REQ-0375 (Priority: 2), Retrieval of postage stamp light curve images: "Postage stamp cutouts, of size **postageStampSize** [51 pixels] square, of all observations of a single Object shall be retrievable within **postageStampRetrievalTime** [10 seconds], with **postageStampRetrievalUsers** [10] simultaneous requests of distinct Objects."
- DMS-REQ-0374 (Priority: 1b), Retrieval of a PVI from a single CCD: "A Processed Visit Image of a single CCD shall be retrievable using the VO SIAv2 protocol within **pviRetrievalTime** [10 seconds] with **pviRetrievalUsers** [20] simultaneous requests for distinct single-CCD PVIs."
- DMS-REQ-0376 (Priority: 1b), Retrieval of focal-plane visit images: "All Processed Visit Images for a single visit that are available in cache, including mask and variance planes, shall be identifiable with a single IVOA SIAv2 service query and retrievable, using the link(s) provided in the response, within **allPviRetrievalTime** [60 seconds]. This requirement shall be met for up to **allPviRetrievalUsers** [10] simultaneous requests for distinct focal-plane PVI sets."
- DMS-REQ-0377 (Priority: 1b), Retrieval of a CCD-sized image from a coadd: "A CCD-sized cutout of a coadd, including mask and variance planes, shall be retrievable using the IVOA SODA protocol within **ccdRetrievalTime** [15 seconds] with **ccdRetrievalUsers** [20] simultaneous requests for distinct areas of the sky."
- DMS-REQ-0373 (Priority: 2), Retrieval of focal-plane-sized images: "A 10 square degree coadd, including mask and variance planes, shall be retrievable using the IVOA SODA protocol within **fplaneRetrievalTime** [60 seconds] with **fplaneRetrievalUsers** [10] simultaneous requests for distinct areas of the sky."


Select Requirements on Types and Content of Images (assembled here for context)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- DMS-REQ-0326 (Priority: 2), Storing Approximations of Per-pixel Metadata: "Image depth and mask information shall be available in a parametrized approximate form in addition to a full per-pixel form."
- DMS-REQ-0024 (Priority: 1a), Raw Image Assembly: "The DMS shall assemble the combination of raw exposure data from all the readout channels from a single Sensor to form a single image for that sensor. The image data and relevant exposure metadata shall be integrated into a standard format suitable for down-stream processing, archiving, and distribution to the user community."
- DMS-REQ-0068 (Priority: 1a), Raw Science Image Metadata: "For each raw science image, the DMS shall store image metadata including at least: i) Time of exposure start and end, referenced to TAI, and DUT1; ii) Site metadata (site seeing, transparency, weather, observatory location); iii) Telescope metadata (telescope pointing, active optics state, environmental state); iv) Camera metadata (shutter trajectory, wavefront sensors, environmental state); v) Program metadata (identifier for main survey, deep drilling, etc.); and vi) Scheduler metadata (visitID, intended number of exposures in the visit).
- DMS-REQ-0069 (Priority: 1a), Processed Visit Images: "The DMS shall produce Processed Visit Images, in which the corresponding raw sensor array data has been trimmed of overscan and corrected for instrumental signature. Images obtained in pairs during a standard visit are combined. *Discussion:* Processed science exposures are not archived, and are retained for only a limited time to facilitate down-stream processing. They will be re-generated for users on-demand using the latest processing software and calibrations. This aspect of the processing for Special Programs data is specific to each program."
- DMS-REQ-0010 (Priority: 1b), Difference Exposures: "The DMS shall create a Difference Exposure from each Processed Visit Image by subtracting a re-projected, scaled, PSF-matched Template Image in the same passband. *Discussion:* Difference Exposures are not archived, and are retained for only a limited time to facilitate Alert processing. They can be re-generated for users on-demand."
- DMS-REQ-0334 (Priority: 1b), Persisting (Level 2) Data Products: "All per-band deep coadds and best seeing coadds shall be kept indefinitely and made available to users.  *Discussion:* This requirement is intended to list all the data products that must be archived rather than regenerated on demand. DMS-REQ-0069 indicates in discussion that Processed Visit Images are not archived. DMS-REQ-0010 indicates in the discussion that Difference Exposures are not archived."
- DMS-REQ-0279 (Priority: 1b), Deep Detection Coadds: "The DMS shall periodicaly create Co-added Images in each of the *u,g,r,i,z,y* passbands by combining all archived exposures taken of the same region of sky and in the same passband that meet specified quality conditions."
- DMS-REQ-0280 (Priority: 1b), Template Coadds: "The DMS shall periodically create Template Images in each of the *u,g,r,i,z,y* passbands that are constructed identically to Deep Detection Coadds, but where the contributing Calibrated Exposures are limited to a range of observing epochs **templateMaxTimespan** [1 year] the images are partitioned by airmass into multiple bins, and where the quality criteria may be different."
- DMS-REQ-0281 (Priority: 1b), Multi-Band Coadds: "The DMS shall periodically create Multi-band Coadd images which are constructed similarly to Deep Detection Coadds, but where all passbands are combined."
- DMS-REQ-0330 (Priority: 2), Best Seeing Coadds: "Best seeing coadds shall be made for each band (including multi-color)."
- DMS-REQ-0335 (Priority: 1b), PSF-Matched Coadds: "One (ugrizy plus multi-band) set of PSF-matched coadds shall be made but shall not be archived."
- DMS-REQ-0338 (Priority: 2), Targeted Coadds: "It shall be possible to retain small sections of all generated coadds."
- DMS-REQ-0106 (Priority: 1b), Coadded Image Provenance: "For each Coadded Image, DMS shall store: the list of input images and the pipeline parameters, including software versions, used to derive it, and a sufficient set of metadata attributes for users to re-create them in whole or in part."
- DMS-REQ-0132 (Priority: 1a), Calibration Image Provenance: "For each Calibration Production data product, DMS shall record: the list of input exposures and the range of dates over which they were obtained; the processing parameters; the calibration products used to derive it; and a set of metadata attributes including at least: the date of creation; the calibration image type (e.g. dome flat, superflat, bias, etc); the provenance of the processing software; and the instrument configuration including the filter in use, if applicable." - We assume that this information is not only to be recorded but also made available to users.
- DMS-REQ-0329 (Priority: 2), All-Sky Visualization of Data Releases: "Data Release Processing shall generate co-adds suitable for use in all-sky visualization tools, allowing panning and zooming of the entire data release."
- DMS-REQ-0379 (Priority: 1b), Produce All-Sky HiPS Map: "Data Release Production shall include the production of an all-sky image map for the existing coadded image area in each filter band, and at least one pre-defined all-sky color image map, following the IVOA HiPS Recommendation.  *Discussion:* The maximum resolution of the image maps is TBD; however, it would be desirable for it to be at least close to the underlying coadded image resolution, in order not to give a poor impression of the data quality. It is possible that the highest-resolution HiPS tiles could be provided on-demand from the LSST cutout service. [...]"


LSP Requirements (LDM-554)
--------------------------

- 



Planned Image Data Products
===========================

DPDD
----

According to the DPDD, the following types of image data products will be provided:

Current Science Pipelines considerations relating to coadds
-----------------------------------------------------------

Compressed Processed Visit Images
---------------------------------

It was realized early in the life of the LSST project that retaining the fully-calibrated single-epoch images, known as Processed Visit Images (PVIs), would be one of the largest single contributions to the storage capacity budget for the project.
In the preliminary-design era, then, it was decided to plan for a tradeoff of computation against storage space, baselining a capability to recreate PVIs on-demand from the (much smaller) raw data.
(The raw images are smaller because a) they are missing variance and mask planes, and b) they have an integer pixel data type and are generally more suitable for lossless compression as a result.)

Subsequently, after the start of construction, the team recognized the existence of common use cases from the transient and variable phenomena science themes that involve requests for stacks of cutouts following an object through time in the survey.
Such requests could not have been addressed in a timely way if all the single-epoch images had to be re-created.
Under :jira:`RFC-325` a proposal was adopted to add the storage of lossy-compressed PVIs to the baseline, with image services to match.

At a technical level, the current tentative approach to this is to define a new Butler dataset type, ``calexp_comp``, which represents PVIs processed through astronomy-quality lossy compression (``calexp`` is the dataset type for the PVIs themselves).
Rubin/LSST image services will be able to return both native and compressed PVIs, and perform cutouts on either type.
Image metadata services will treat them as separate datasets differing primarily only in their ``dataproduct_subtype`` and their size.


Use Cases
=========


Community Standards
===================

IVOA
----

The planned image service architecture is based on the following core standards:

Endpoint services
^^^^^^^^^^^^^^^^^

Image metadata query:
  ObsTAP, from ObsCore 1.1 or later (as an initial priority), with SIAv2 (also ObsCore-based and serving equivalent data) later

Catalog query:
  TAP 1.1 or later

Image cutout service:
  SODA 1.0 or later

DataLink *links* service:
  DataLink 1.0 or later (very likely to be revised in the near future)

Note that there is no general "image service" *per se* in the current IVOA model.
SIAv2 (and its predecessor) are image *metadata* services, operating on a model of queries returning URLs to the actual images (possibly indirected through DataLink *links* services).
The endpoints described by those URLs need not provide anything other than static file service.
The SODA standard covers the specific case of cutouts (in general *not* previously existing) from identified datasets which themselves may or may not be materialized or statically accessible.

Supporting standards
^^^^^^^^^^^^^^^^^^^^

Service endpoints, self-description, and status:
  VOSI 1.1 or later

Service parameters and datatype definitions:
  DALI 1.1 or later

Asynchronous service interfaces:
  UWS 1.1 or later

Query language:
  ADQL 2.00 or later (ADQL 2.1 is imminent)

Extensibility
^^^^^^^^^^^^^

Where these standards fall short of meeting the needs of the project:

- Many of them have specified avenues for enhancing a service with additional metadata or parameters.  Examples:

  - A TAP/ObsTAP service can supply additional information in its `TAP_SCHEMA` schema;

  - A DataLink *links* service can provide publisher-specific values for the `semantics` column; and

  - An ObsCore table may contain a) optional ObsCore-defined columns, and/or b) additional service-specific columns.

- The project is free to develop service protocols and data formats of its own.
  However, consistent with the "VO-first" plan, we will strive to respect the layering of standards.
  For example, if the creation of a non-IVOA-standard service is required in order to fulfill a need, it should still be compliant with DALI, VOSI, and, if applicable, UWS.
  This policy facilitates service concepts such as the use of DataLink to direct users to these non-standard services.

Service Registration
^^^^^^^^^^^^^^^^^^^^

The present edition of this document does not address how Rubin/LSST services will be represented in the IVOA Registry/ies, nor does it address whether the project will operate its own registry.

CAOM2
-----

FITS
----

WCS Metadata
^^^^^^^^^^^^

Image files produced in the FITS format by the project will contain FITS-standard WCS information.
For coadded images, which are warped to a precise pixel grid, a trivial TAN projection will be used.
For FITS tiles in the HiPS image maps produced by the project, the HPX projection will be used.

For single-epoch images where the actual Camera WCS must be modeled, the full-precision WCS model produced by the pipeline will not be completely captured by FITS-standard WCS headers.
Full-precision WCS data will be included in an extension in the image file containing WCS information, using the ASDF model also used by JWST, represented as YAML ASCII string data in a binary table extension.

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

TAP Service for Additional Observation Metadata
-----------------------------------------------

SIAv2 Service
-------------

SODA Service(s)
---------------

HiPS Service
------------

Unlike the other services discussed above, a standards-conformant HiPS service is not defined within the DALI/VOSI framework and does not have to have any server-side intelligence; it is perfectly acceptable for a HiPS service to be a static file-access service, with URLs directly mapped onto a physical directory tree on the server.
A HiPS service need not provide an `/availability` endpoint and has no well-defined way to provide a `/capabilities` endpoint even if the publisher so desired.

A HiPS service *may* satisfy the file-access expectations of the standard by materializing the files at run time instead of serving them from a static tree.
However, in the present proposal we do not expect to do so.

Availability Endpoints
----------------------

Contemporary IVOA-standard services based on DALI and VOSI are generally expected to each have an `/availability` endpoint.
Logically, in a production scenario, a service cannot completely cover the space of availability reporting, including all forms of downtime, entirely on its own.
It can report itself as down when the service itself is operating normally but is unable to reach other services or resources on which it transitively depends, but it cannot in general always report on its own downtime.

In order to address this, we propose an architecture in which the `/availability` endpoints of all services are redirected, for example at the Kubernetes ingest-rule level, to a lightweight central "availability status" dashboard service, which, ideally, could be maintained "up" even during major datacenter maintenance, to allow serving the "down" status of the actual services, and an expected return-to-service time if available.
This service should of course return VOSI-compatible results.

It might then be appropriate for the "dashboard" service to call through to hidden `/availability` endpoints of the individual services to obtain any "down" indications from them that arise from transitive service-dependency issues.
However, this will require more detailed analysis.

The Rubin Science Platform landing page should make available a "red/green" status display based on the "dashboard" services.
One possible implementation might be a "summary" status that is always displayed, with the ability to click through on it to a more detailed status page.


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


Major Open Questions
====================

IVOA Registry Representation
----------------------------

As noted above, the present proposal does not address this issue.

HiPS Discovery
--------------

As noted above, we currently propose providing two endpoints for each map, serving the same "upper" (lower-resolution) layers of the HiPS map, with the highest-resolution layer(s) only in the endpoint which requires data rights for access.
Project requirements call for the HiPS maps from the LSST survey to be registered with the CDS HiPS registry.
The public-access map should be registered with `hips_status = public master clonableOnce`, to support the HiPS model for replication of openly accessible datasets.
The data-rights-limited map should be registered with `hips_status = private master unclonable`.
Pending future developments that might change this decision, the replication of the limited-access data should be carried out through the larger process of establishment of "independent Data Access Centers" (IDACs) and distribution of data to them, rather than through the HiPS replication mechanism.

MOC Discovery
-------------


Appendices
==========

Appendix: Implementation Notes
------------------------------

While the principal purpose of this note is to define the service-level architecture, there are some implementation details that have been developed as a proposal alongside that architecture, and that are worth documenting.

We address the non-TAP image-related services first.
A short discussion of some relevant TAP issues follows.

Generic DALI/VOSI service implementation framework
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The service concept described above involves the creation of a variety of services, some directly conformant to application-level IVOA standards such as SODA and DataLink, and some unique to Rubin but conformant to the framework standards such as DALI and VOSI.
In order to ease the development, deployment, and maintenance of these services, we envision the creation and/or re-use of a framework for supplying these common elements of a service implementation.

Several of the envisioned services (SODA/cutouts, virtual-data-product re-creation, forced photometry on demand) ultimately require the invocation of Rubin Science Pipelines stack code, with its Python interfaces.
Hence we would need either a service framework actually implemented in Python, or one implemented in another language such as Java or its descendants which is then able to invoke a "payload" of Python code.

The service framework should provide, at a minimum:

- skeleton implementations of `/availability` and `/capability` endpoints, with an extensible interface for supplying service-application-specific behavior where needed; and
- definition and parsing of a URL-with-query-parameters API, following DALI standards, including the parsing of DALI-compliant parameter value types and their translation into Python objects, ideally Astropy objects where available.

Desirable features would include:

- the ability to generate DataLink-compatible service descriptors from the API parameter definitions; 
- the ability to either generate, or start from, an OpenAPI v3 service description, enabling the provision of "Swagger" UI pages for users to explore the APIs' capabilities, and perhaps in the future to facilitate the provision of metadata-driven widgets in the Notebook Aspect for access to the APIs.

For an example of the use of the OpenAPI specification and Swagger to deliver an elementary UI, see the CADC TAP service page at https://www.cadc-ccda.hia-iha.nrc-cnrc.gc.ca/argus/ and the SIA service page at https://www.cadc-ccda.hia-iha.nrc-cnrc.gc.ca/sia/ .
The value of this for user education and for operations-team debugging activities is unmistakable.
There is a cost in providing interfaces of this sort, but it is greatly mitigated by building the necessary apparatus in a generic way.

**Some** aspects of this can likely be obtained on the Java side by the adoption of layered elements of the CADC code base.
However, the propagation of DALI parameters to Python code that actually performs computational work is an important part of the vision.
On this side, there is, at the time of writing, no fully satisfactory open-source Python framework with this features.
It would be immediately useful internally, as well as a valuable community service, if the Rubin project were to produce such a framework.

Such a framework would be designed to minimize the "boilerplate" work on API parsing and DALI/VOSI-conformance associated with delivering a new service.


UWS-based asynchronous services
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


VOTable annotation
^^^^^^^^^^^^^^^^^^


TAP (and ObsTAP)
^^^^^^^^^^^^^^^^

The TAP service planned for use in the Science Platform is the CADC-derived, Java-based one which is already in use in the prototype/demonstration-oriented Science Platform deployed at NCSA.
Thus far the CADC code base has proven valuable.
It was relatively easily adapted to the Qserv back end, and the relationship between the Rubin project and CADC is positive and has allowed upstreaming a number of changes.

The Rubin project has not yet attempted to use the currently limited capabilities of the CADC framework for DataLink service descriptor or `<GROUP>` annotation of VOTable query results.
Because DataLink is an important part of the service concept described in this note, we may find that it is an area in which it is valuable for us to make a substantial upstream contribution.

SIAv2
^^^^^

As also noted in the main body of this document, CADC has an implementation of SIAv2 which performs virtually all of the actual work of performing a query by deferring to an underlying ObsTAP service.
This is feasible because of the close alignment of the ObsCore (which contains ObsTAP) and SIAv2 standards, in which almost all SIAv2 query capabilities are expressible as ADQL against the ObsCore data model.
We should investigate whether the CADC implementation is in fact practical for use against Rubin's specific ObsTAP service.

Note that an implementation of SIAv2 as a call-through to ObsTAP would then directly benefit from any progress on DataLink and/or `<GROUP>` annotations in the VOTable output from the ObsTAP service.



Appendix: Portal-derived Requirements on Image Services
-------------------------------------------------------

The LDM-554 requirements on Portal Aspect functionality include a number of requirements on capabilities for image and image metaata queries.
While these requirements do not specify an implementation for the interface with any underlying services used for image data retrieval, the basic architecture of the three-Aspect Science Platform implies that data access capabilities in the Portal Aspect (and Notebook Aspect) should be paralleled by capabilities in the Web API Aspect.

In this Appendix we list the relevant Portal Aspect requirements and discuss their implications on Web API Aspect image and image-metadata query services.

Note that many of the capabilities mentioned below will not be part of the "frozen" post-DM-10 Portal Aspect.
However, if a future Portal Aspect implementation is to provide these capabilities, it should be understood in advance what Web API Aspect capabilities would have to be available for the Portal Aspect capabilities eventually to be implemented.

Requirements
------------

- DMS-PRTL-REQ-0035, Query for Single Epoch Visit Images: "The Portal aspect shall enable a user to proceed from a visit-selection query or a list of visits and return a list of all single-epoch images of a specified type corresponding to those visits.
*Discussion:* The common image types will be raw, PVI (processed, i.e., calibrated, visit image), and difference image."

This requirement suggests that the Web API Aspect must provide an interface for querying image data by a visit ID, or list of them, and by a recognized image type.

- DMS-PRTL-REQ-0036, Query for Single Epoch Raft Images: "The Portal aspect shall enable a user to limit the list of images selected by a single-epoch visit image query to those from a specified raft."

This requirement suggests that either the Web API Aspect must provide an additional query-restriction capability that takes a raft ID (or CCD ID, see below), or there must be a straightforward way for the Portal Aspect to apply a raft ID restriction on the results of a single-visit image query.

- DMS-PRTL-REQ-0037, Query for Single Epoch CCD Image: "The Portal aspect shall enable a user to limit the list of images selected by a single-epoch visit image query to those from a specified CCD."

The same considerations apply here as to the raft-ID-based query above.
Note, however, that post-hoc limitations of visit queries to one out of 189 query results (i.e., by CCD ID) is a fairly inefficient process.
This suggests that it may be preferable to provide a server-side capability for restricting queries by CCD ID.
This appears to be consistent with the Butler Gen3 data model and the plan to persist images on the back end in CCD-sized files.

- DMS-PRTL-REQ-0038, Single-Epoch Image Query Specifications: "The Portal aspect shall provide UI support for queries for visits and their single-epoch images of specified type, based on image metadata parameters including pointing, time and date, and filter selection, as well as on parameters from the Reformatted EFD.
*Discussion:* The parameters specifically named are expected to be highlighted in the UI, rather than requiring the user to scroll through a long generic-table-query form to find the appropriate fields.
The UI will provide support for generating a join query including tables from the R-EFD, and for selecting the R-EFD tables and columns to use."

An efficient Portal Aspect implementation here depends on the ability to perform these complex queries on the server side.
This appears to require the ability to perform ``JOIN`` queries between the primary image and visit metadata table(s) and EFD tables, which must therefore be on the same TAP service and, most likely, on the same underlying database service (though outside-the-database JOINs at the TAP layer are a possible technical alternative).

(INCOMPLETE - high-water-mark of completed work on this Appendix)

- DMS-PRTL-REQ-0039, Coadded Image Query Specifications: "The Portal aspect shall provide UI support for queries for coadded images based on the image metadata that describe the provenance of the images (e.g., filters, position on the sky, date, number of single-epoch images, coverage, survey depth)."




Service Implications
--------------------




.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
