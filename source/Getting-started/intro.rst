Getting started
================

Penglai-Enclave Overview
-------------------------
Penglai Enclave is an open-sourced enclave system an RISC-V, which is suitable for  multiple scenarios.

There are two main series of Penglai enclaves:

+ **Penglai-TVM:** 
  It realizes a scalable enclave system with the pure software implementation of Guarded page table (details of guarded page table can be found in our `OSDI paper <https://ipads.se.sjtu.edu.cn/zh/publications/FengOSDI21-preprint.pdf>`_). 
  With the benefit of our scalable design, Penglai-TVM can realize the scalable memory protection, zero-copy communication and enclave fast boot. You can get the code Penglai-TVM from `here <https://github.com/Penglai-Enclave/Penglai-Enclave-TVM>`_ .

+ **Penglai-PMP**
  It leverages the PMP/sPMP to protect enclave memory. sPMP is our new hardware extension, which will act as RISC-V MPU later (you can find more information in sPMP `white paper <https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP/blob/master/Proposal-of-sPMP.pdf>`_).
  Penglai-PMP version is more lightweight and can be used in the IoT device. You can get the code of Penglai-PMP from `here <https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP>`_ .

Quick start
------------
Penglai enclave can support multiple platforms:

.. toctree::
    :maxdepth: 1
 
    Running-Penglai-TVM-with-QEMU.rst
    Running-Penglai-TVM-with-FPGABoard.rst

