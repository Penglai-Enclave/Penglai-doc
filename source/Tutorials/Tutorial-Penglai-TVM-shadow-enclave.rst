Shadow enclave (Penglai-TVM)
==============================

This tutorial introduces the server enclave in Penglai. Server enclave is a special enclave type, it will not run until other enclaves call it.
With the server enclave, we can implement an enclave chain in the Penglai. Server enclave can act as a separate library or OS server, etc.

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
  fork-host

Test scripts
>>>>>>>>>>>>>

::

  sh install.sh
  ./fork-host hello-world

Host app
----------

.. code-block:: c

    struct elf_args* enclaveFile = malloc(sizeof(struct elf_args));
    char * eappfile = argv[1];
    elf_args_init(enclaveFile, eappfile);

    if(!elf_valid(enclaveFile))
    {
        printf("error when initializing enclaveFile\n");
        goto out;
    }

    struct PLenclave* enclave = malloc(sizeof(struct PLenclave)); 
    struct enclave_args* params = malloc(sizeof(struct enclave_args));
    PLenclave_init(enclave);
    enclave_args_init(params);
    //Mark the enclave as a shaodw enclave
    params->type = SHADOW_ENCLAVE;

    if(PLenclave_create(enclave, enclaveFile, params) < 0 )
    {
        printf("host: failed to create enclave\n");
    }

We propose the `shadow enclave <https://github.com/Penglai-Enclave/Penglai-Enclave-sPMP>`_ to boost the enclave initialization.
Creating a shadow enclave is similar to creating a normal enclave, just set enclave type in the creation parameter to the ``SHADOW_ENCLAVE``.
After creating a shadow enclave, it will return its eid. A forked enclave can use this shadow enclave eid to initialize an enclave instance. 

.. code-block:: c

    enclave->fd = fd;
    enclave->user_param.eid = eid;
    enclave->eid = eid;
    PLenclave_run(enclave)

Host can fork multiple enclave instances from a single shadow enclave. Each enclave instance will not interfere each other, with its own stack and heap.

Fork a new enclave instance from shadow enclave can reduce the measurement overhead, which is suitable in the fast-boot and auto-scale scenarios.