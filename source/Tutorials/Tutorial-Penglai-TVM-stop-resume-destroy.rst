Stop, Resume and Destroy (Penglai-TVM)
========================================

This tutorial introduces how to stop resume or destroy enclave. Host can stop or destroy an enclave compulsively. The DOS attack triggered by the untrusted host is out of scope.

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

.. attention::
            
    ``PLenclave_destroy`` can also destroy the *server enclave* and *shadow enclave*. Before destroying these special types of enclaves, you need to ensure that server enclaves or shadow enclaves will never be invoked later.
    
    ``PLenclave_stop`` and ``PLenclave_resume`` can not be used to stop / resume *server enclave* or *shadow enclave*, as they are not the runnable instances.

