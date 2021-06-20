Stop, Resume and Destroy (Penglai-TVM)
========================================

This tutorial introduces how to stop resume or destroy enclave. Host can stop or destroy an enclave compulsively. The DOS attack triggered by the untrusted host is out of scope.

Prerequisite
-------------
Before running this tutorial, please make sure you have finished :doc:`getting started <../Getting-started/intro>`.

In current veriosn, we use ramfs as the rootfs, so it needs to build all the files into ramfs (using buildroot). 
To simplify this process, we define a macro ``SDK_FILES``, all the files defined in ``SDK_FILES`` will be added in the initramfs.
So when machine boots, all these files will be existed in the root.

Make sure all the requested files in this tutorial are added in the ``SDK_FILES``. 

Required files
>>>>>>>>>>>>>>>

::

  penglai.ko
  install.sh
  test-stop
  loop

Test scripts
>>>>>>>>>>>>>

::

  sh install.sh 
  ./test-stop loop

Host app
----------

.. code-block:: c

  // Another host thread
  // Stop a running enclave
  int PLenclave_stop(struct PLenclave *PLenclave);
  // Resume a stopped enclave
  int PLenclave_resume(struct PLenclave *PLenclave);
  // Destroy an enclave
  int PLenclave_destroy(struct PLenclave *PLenclave);

When an enclave is running, another host thread can stop the running enclave, using ``PLenclave_stop`` interface. The running enclave will be interrupted and monitor will store its context.
Host can resume the stopped enclave using the ``PLenclave_resume``, the enclave will continue to run from the last stop point. 
Host can destroy an enclave and reclaim its resource using ``PLenclave_destory``.
Host can not destroy an enclave which is already exited.
