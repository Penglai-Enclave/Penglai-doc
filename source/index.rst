.. Penglai-Enclave documentation master file, created by
   sphinx-quickstart on Mon Jun  7 18:25:43 2021.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to Penglai-Enclave's documentation!
===========================================

Penglai is a set of security solutions based on Trusted Execution Environment.
This website contains an overview of the whole project.

It currently supports RISC-V platforms, including both high-performant MMU RISC-V64 arch and MCU (RISC-V32, no MMU).

Systems
--------

Penglai contains a set of systems satisfying different scenarios.

+ Penglai-TVM: it is based on OpenSBI, supports fine-grained isolation (page-level isolation) between untrusted host and enclaves. The code is maintained in Penglai-TVM_.

+ Penglai-sPMP/PMP: it can utilize either PMP or our sPMP (S-mode PMP) proposal to provide basic enclave functionalities. Refer Penglai-sPMP_ for more info. A version based on OpenSBI for Nuclei devices is maintained in Nuclei-SDK_.

+ Penglai-MCU: it supports Global Platform, and PSA now. Not open-sourced. Refer Penglai-MCU_ for more info.


.. _Penglai-TVM: https://github.com/Penglai-Enclave/Penglai-Enclave-TVM
.. _Penglai-MCU: https://github.com/Penglai-Enclave/#
.. _Nuclei-SDK: https://github.com/Nuclei-Software/nuclei-linux-sdk/tree/dev_flash_penglai_spmp
.. _sPMP: https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP
.. _Penglai-sPMP: https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP


.. toctree::
   :maxdepth: 1
   :caption: Getting Started

   Getting-started/intro.rst

.. toctree::
   :maxdepth: 1
   :caption: Tutorials

   Tutorials/Tutorial-index.rst
   

.. toctree::
   :maxdepth: 1
   :caption: Penglai manual

   Penglai-manual/Penglai-Opensbi-Extension-API
   Penglai-manual/User-Manual-TVM
   Penglai-manual/User-Manual-PMP

.. toctree::
   :maxdepth: 1
   :caption: FAQ

   Getting-started/FAQ

