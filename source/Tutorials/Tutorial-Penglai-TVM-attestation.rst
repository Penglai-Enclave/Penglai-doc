Attestation (Penglai-TVM)
==============================

Penglai supports local attestation, required by host and enclave. Monitor will return the attestation report, including the enclave and secure monitor's measurement.


Prerequisite
-------------
Before using this tutorial, please make sure you have finished :doc:`getting started <../Getting-started/intro>`.

In current veriosn, we use ramfs as the rootfs, so it needs to build all files into ramfs previously. 
To simplify this process, we define a macro ``SDK_FILES``, all the files defined in ``SDK_FILES`` will be added to the initramfs during the build phase.
When machine boots, all these files are existed in the **root** directory.

Make sure all the requested files in this tutorial are added in the ``SDK_FILES``. 

Required files
>>>>>>>>>>>>>>>

::

  penglai.ko
  install.sh
  test-caller
  caller
  server
  server1

Test scripts
>>>>>>>>>>>>>

::

  sh install.sh
  ./test-caller caller server server1

Get the attestation report in the host
----------------------------------------

.. code-block:: c

  struct sm_report_t
  {
    unsigned char hash[HASH_SIZE];
    unsigned char signature[SIGNATURE_SIZE];
    unsigned char sm_pub_key[PUBLIC_KEY_SIZE];
  };

  struct enclave_report_t
  {
    unsigned char hash[HASH_SIZE];
    unsigned char signature[SIGNATURE_SIZE];
    uintptr_t nonce;
  };

  struct report_t
  {
    struct sm_report_t sm;
    struct enclave_report_t enclave;
    unsigned char dev_pub_key[PUBLIC_KEY_SIZE];
  };

  int PLenclave_attest(struct PLenclave *PLenclave, uintptr_t nonce);

 
Host can call the ``PLenclave_attest`` function to get the enclave attestation report. The attestation report is composed of device public key, secure monitor report and enclave report.
Host can authenticate a legal enclave with the attestation report. Device public key and secure monitor public key are used to verify corresponding signatures.
Nonce is required in the attestation to ensure others can not reuse a stale attestation report. 

Get the attestation report in the enclave
-------------------------------------------

Enclave can get the attestation report of itself and other :doc:`server enclaves <Tutorial-Penglai-TVM-server-enclave>`, using the following interface:

.. code-block:: c

  int get_report(char* name, struct report_t *report, unsigned long nonce);
  get_report(NULL, report, 1);
  get_report("server-enclave", report, 1);

Enclave can acquire it attestation report using the ``get_report()``. The structure of the attestation report is same as the report used by host.
If the attested enclave name is set to NULL, it will return the attestation report of current enclave, otherwise, it will return the attestation report with the given server enclave name.
Before invoking a server enclave, it is better to verify the server enclave measurement with ``get_report``.

Remote attestation can be realized based on the local attestation. 