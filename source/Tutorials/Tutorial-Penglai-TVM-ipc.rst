IPC for enclave-enclave and enclave-host (Penglai-TVM)
========================================================

This tutorial introduces how to implement the IPC between the enclave-enclave and enclave-host in the Penglai-TVM. 
Basically, there are two methods to implement the IPC: the shared buffer and ownership transfer. We will introduce both these two methods in the tutorial.

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

    // Allocate the shared memory for enclave
    unsigned long shm_size = 0x1000 * 4;
    int shmid = PLenclave_shmget(shm_size);
    void* shm = PLenclave_shmat(shmid, 0);

    //Write the content into shared memory
    // memcpy(shm, CONTENT, shm_size);

    caller_params->shmid = shmid;
    caller_params->shm_offset = 0;
    caller_params->shm_size = shm_size;

    // Allocate the relay/schrodinger page for enclave
    unsigned long mm_arg_size = 0x1000 * 4;
    int mm_arg_id = PLenclave_schrodinger_get(mm_arg_size);
    void* mm_arg = PLenclave_schrodinger_at(mm_arg_id, 0);

    // Write the content into the shrodinger/relay page (ownership transferred memory)
    // memcpy(mm_arg, CONTENT, mm_arg_size);

    // Enclave needs a inde
    char enclave_name[15];
    sprintf(enclave_name, "test-caller");
    strcpy(caller_params->name, enclave_name);

    if(PLenclave_create(caller_enclave, caller_enclaveFile, caller_params) < 0 )
    {
        printf("host: failed to create caller_enclave\n");
        goto out;
    }

    // Assign the schrodinger page to the enclave before it runs
    if(mm_arg_id > 0 && mm_arg)
        PLenclave_set_mem_arg(caller_enclave, mm_arg_id, 0, mm_arg_size);

    PLenclave_run(caller_enclave);


There are two method for enclave-host communication: shared memory and ownership transfer. 

Shared memory
>>>>>>>>>>>>>>

As for shared memory, we provide following interfaces to allocate and assign shared memory for the host:

.. code-block:: c
  
    int PLenclave_shmget(unsigned long size);
    void* PLenclave_shmat(int shmid, void* addr);
    int PLenclave_shmdt(int shmid, void* addr);
    int PLenclave_shmctl(int shmid);

+ ``PLenclave_shmget`` Get the shared memory id with the given size. Each id binds with a range of shared memory.
+ ``PLenclave_shmat`` Get the corresponding shared memory address with the given shared memory id, and maps shared memory into host address space.
+ ``PLenclave_shmdt`` Unmap the shared memory from the host address space.
+ ``PLenclave_shmctl`` Revoke the shared memory id, no one can retrieve the shared memory with this id.

Shared memory will be mapped in both host and enclave address space. So host and enclave can use this memory simultaneously.

There are several enclave, parameters related with the shared memory, see in the below:

.. code-block:: c

    enclave_params->shmid = shmid;
    enclave_params->shm_offset = 0;
    enclave_params->shm_size = shm_size;

Host needs to set these parameters, before creating an enclave.

Ownership transferred memory
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

As for ownership transferred memory. We name the ownership transferred memory as the ``schrodinger page`` for host and ``relay page`` for enclave.
We provide following interfaces to allocate and assign schrodinger pages for the host:

.. code-block:: c
  
    int PLenclave_schrodinger_get(unsigned long size);
    void* PLenclave_schrodinger_at(int id, void* addr);
    int PLenclave_schrodinger_dt(int id, void* addr);
    int PLenclave_schrodinger_ctl(int id);

+ ``PLenclave_schrodinger_get`` Get the schrodinger page id with the given size. Each id binds with a range of schrodinger pages.
+ ``PLenclave_schrodinger_at`` Get the corresponding schrodinger page address with the given schrodinger page id, and maps schrodinger page into host address space.
+ ``PLenclave_schrodinger_dt`` Unmap the schrodinger page from the host address space.
+ ``PLenclave_schrodinger_ctl`` Revoke the schrodinger page id, no one can retrieve the schrodinger page with this id.

Schrodinger page can only be mapped in either host or enclave. When schrodinger pages are allocated, they are first mapped in the host space. So host can read / write these pages as normal memory. 
When the enclave runs, monitor will guarantee that all the schrodinger pages is unmaped in the host, and remap to enclave. So enclave can read / write these pages.
Schrodinger pages are used to defend against the Time-Of-Check-To-Time-Of-Use (TOCTTOU) attack, and can realize the zero-copy communication.

.. code-block:: c

    PLenclave_set_mem_arg(enclave, mm_arg_id, 0, mm_arg_size);

Host can use this function to bind the schrodinger page with given enclave.

Enclave app
------------

Host-Enclave IPC
>>>>>>>>>>>>>>>>>

.. code-block:: c
  
    // Get the content in the shared memory
    int *shm = (int *)args[10];
    unsigned shm_size = (unsigned long) args[11];

    // Get the content in the ralay/schrodinger page (zero copy) 
    int *relay_page = (int *)args[13];
    unsigned relay_page_size = (unsigned long) args[14];

You can get the shared memory and relay page (schrodinger page) in the corresponding registers.

``a0`` and ``a1`` are reserved for shared memory (shared memory bases address and size), and ``a3`` and ``a4`` are reserved for relay page (relay page base address and size).
Enclave can use this memory directly

Enclave-Enclave IPC
>>>>>>>>>>>>>>>>>>>>

Simialr as the Host-Enclave IPC. Enclave-Enclave IPC can also use two methods: one is shared memory and another is ownership transferred (relay page in enclave).

As for ``shared memory``, host can assign a single shared memory to multiple enclaves. So, these enclaves can share the same memory,

As for ``relay page``, we define an IPC structure in enclave library.

.. code-block:: c

    struct call_enclave_arg_t
    {
    unsigned long req_arg;
    unsigned long resp_val;
    unsigned long req_vaddr;
    unsigned long req_size;
    unsigned long resp_vaddr;
    unsigned long resp_size;
    };

In this IPC structure, we define the request and response parameters. As for the request parameters, it can be passed in the register or the transferred memory.
The format of the response parameters are similar to the request parameters, supporting return register and transferred memory. The transferred memory used in the enclave can be allocated with the following interface:

.. code-block::

   req_vaddr = eapp_mmap(NULL, size);

We also define the enclave call in the eapp library, which can support an synchronous IPC call for a :doc:`server enclave <Tutorial-Penglai-TVM-server-enclave>`.
The caller enclave will wait until the callee return back.
IPC structure is the calling parameter, and can transfer between the caller and callee enclaves.
The transferred memory defined in the IPC structure will change its ownership and remap to the destined enclave.
This mechanism ensures that only one enclave can access this memory range.

.. code-block::

    struct call_enclave_arg_t call_arg;
    call_arg.req_arg = arg0;
    call_arg.req_vaddr = req_vaddr;
    call_arg.req_size = size;
    call_enclave(server_handle, &call_arg);

You can find more details of server enclave and how to implement a IPC between caller enclave and server enclave in :doc:`tutorial-penglai-tvm-server-enclave <Tutorial-Penglai-TVM-server-enclave>`
