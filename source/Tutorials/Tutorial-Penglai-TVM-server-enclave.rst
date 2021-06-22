Server enclave (Penglai-TVM)
==============================


This tutorial introduces the server enclave in Penglai. Server enclave is a special type of enclave, it will not run until other enclaves call it.
With the server enclave, we can implement an enclave chain in the Penglai. Server enclave can act as a separate library or OS server, etc.

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

Host app
----------

.. code-block:: c

    server:
        server_enclaveFile = malloc(sizeof(struct elf_args));
        char *server_eappfile = argv[2];
        elf_args_init(server_enclaveFile, server_eappfile);

        if(!elf_valid(server_enclaveFile))
        {
            printf("host:error when initializing server_enclaveFile\n");
            goto out;
        }

        server_enclave = malloc(sizeof(struct PLenclave));
        server_params = malloc(sizeof(struct enclave_args));
        PLenclave_init(server_enclave);
        enclave_args_init(server_params);
        strcpy(server_params->name, "test-server");
        server_params->type = SERVER_ENCLAVE;

        if(PLenclave_create(server_enclave, server_enclaveFile, server_params) < 0)
        {
            printf("host:failed to create server_enclave\n");
            goto out;
        }

The creation of server enclave is similar to the normal enclave. To create a server enclave, you only need to set the enclave type in enclave creating parameter as ``SERVER_ENCLAVE``, and assign the server enclave with a unique name.
The enclave name is used as the identification when calling this server enclave. 

.. code-block::c

  server_params->type = SERVER_ENCLAVE;
  strcpy(server_params->name, "test-server");

Host does not need to run a server enclave, and actually, the server enclave cannot run alone. Server enclave can be called by other enclaves or server enclaves, and reuse their host context.

.. attention::

    Currently, the server enclave can only be called sequentially. We will support the concurrent server enclave invocation later. 

Host can destroy a server enclave when server enclave is no longer needed, using the following API:

.. code-block:: c
 
  PLenclave_destroy(server1_enclave);


Enclave app
-------------

Caller enclave
>>>>>>>>>>>>>>>>>

.. code-block:: c
  
    unsigned long server_handle = acquire_enclave(server_name); 
    struct call_enclave_arg_t call_arg;
    call_enclave(server_handle, &call_arg);

In the caller enclave, it first needs to acquire the server enclave handler with its enclave name. After retrieving the server enclave handler, a caller enclave can call this server enclave and pass the IPC structure. You can find more details of IPC between enclave-enclave in :doc:`tutorial-penglai-tvm-ipc <Tutorial-Penglai-TVM-ipc>`.
Caller enclave will wait until the server enclave returns.

Callee enclave
>>>>>>>>>>>>>>>>

.. code-block:: c

    unsigned long caller_arg0 = args[10];
    void* caller_vaddr = (void*)args[11];
    unsigned long caller_size = args[12];

As for server enclave, it can retrieve the caller IPC parameters in the registers. ``a0`` reserves the IPC parameter stored in the register. ``a1`` and ``a2`` indicate the transferred memory range defined in IPC structure.
Server enclave can access this memory range directly. 

When server enclave returns, it also needs to define a return IPC structure, and invokes SERVER_RETURN to return to the caller enclave.

.. code-block:: c

    ret_arg.resp_vaddr = vaddr;
    ret_arg.resp_size = size;
    ret_arg.resp_val = value;
    SERVER_RETURN(&ret_arg);

Caller enclave can receive the return IPC structure and continue to run.