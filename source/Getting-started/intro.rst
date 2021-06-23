Getting started
================

Penglai-Enclave Overview
-------------------------
Penglai Enclave is an open-sourced enclave system an RISC-V, which is suitable for  multiple scenarios.

There are two main series of Penglai enclaves:

+ **Penglai-TVM:** 
  It realizes a scalable enclave system with the pure software implementation of Guarded page table (details of guarded page table can be found in our `OSDI paper <https://ipads.se.sjtu.edu.cn/zh/publications/FengOSDI21-preprint.pdf>`_). 
  With the benefit of our scalable design, Penglai-TVM can realize the scalable memory protection, zero-copy communication and enclave fast boot. You can get the source code of Penglai-TVM `here <https://github.com/Penglai-Enclave/Penglai-Enclave-TVM>`_ .

+ **Penglai-PMP**
  It leverages the PMP/sPMP to protect enclave memory. sPMP is our new hardware extension, which will act as RISC-V MPU later (you can find more information in sPMP `white paper <https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP/blob/master/Proposal-of-sPMP.pdf>`_).
  Penglai-PMP version is more lightweight and can be used in the IoT devices. You can get the source code of Penglai-PMP `here <https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP>`_ .

Quick start
------------
Penglai enclave can support multiple platforms:

.. toctree::
    :maxdepth: 1
 
    Running-Penglai-TVM-with-QEMU.rst
    Running-Penglai-TVM-with-FPGABoard.rst
    Running-Penglai-PMP-with-QEMU.rst

Tutorial
----------
The tutorials for Penglai Enclave, which introduce the enclave usages and basic functionalities.

* :doc:`Tutorials <../Tutorials/Tutorial-index>`

User manual
------------
In the user manual, it defines all interfaces and functions you can invoke in host application and enclaves.
More details can be found in the following documents:

* :doc:`Penglai-TVM-Manual </Penglai-manual/User-Manual-TVM>`
* :doc:`Penglai-PMP-Manual </Penglai-manual/User-Manual-PMP>`

FAQ
----

* :doc:`Frequently Asked Questions <./FAQ>`



