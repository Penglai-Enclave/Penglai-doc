Hello World (Penglai-TVM)
===========================

This tutorial introduces how to run a hello-world demo in the Penglai-TVM. 
It requires two parts to run an enclave: host app and enclave app. Host app, is the untrusted part running in the host kernel, which can create or run an enclave.
Enclave app, is the trusted part running in the enclave.

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
  hello-world
  host

Test scripts
>>>>>>>>>>>>>

::

  sh install.sh
  ./host hello-world


  

Host app
----------
.. code-block:: c

    #include "penglai-enclave.h"
    #include <stdlib.h>
    #include <pthread.h>

    struct args
    {
        void* in;
        int i;
    };

    void* create_enclave(void* args0)
    {
        struct args *args = (struct args*)args0;
        void* in = args->in;
        int i = args->i;
        int ret = 0, result = 0;
        
        struct PLenclave* enclave = malloc(sizeof(struct PLenclave));
        struct enclave_args* params = malloc(sizeof(struct enclave_args));
        PLenclave_init(enclave);
        enclave_args_init(params);

        struct elf_args *enclaveFile = (struct elf_args *)in;
        
        if(PLenclave_create(enclave, enclaveFile, params) < 0)
        {
            printf("host:%d: failed to create enclave\n", i);
        }
        else
        {
            while (result = PLenclave_run(enclave))
            {
                printf("[ERROR] host: result %d val is wrong!\n", result);
                goto free_enclave;
            }
            printf("host: exit enclave is successful \n");
        }
        printf("host: PLenclave run is finish \n");

    free_enclave:  
        free(enclave);
        free(params);

        pthread_exit((void*)0);
    }

    int main(int argc, char** argv)
    {
        if(argc <= 1)
        {
            printf("Please input the enclave ELF file name\n");
        }

        int thread_num = 1;
        if(argc == 3)
        {
            thread_num = atoi(argv[2]);
            if(thread_num <= 0)
            {
            printf("error number\n");
            return -1;
            }
        }

        pthread_t* threads = (pthread_t*)malloc(thread_num * sizeof(pthread_t));
        struct args* args = (struct args*)malloc(thread_num * sizeof(struct args));

        struct elf_args* enclaveFile = malloc(sizeof(struct elf_args));
        char* eappfile = argv[1];
        elf_args_init(enclaveFile, eappfile);
        
        if(!elf_valid(enclaveFile))
        {
            printf("error when initializing enclaveFile\n");
            goto out;
        }

        for(int i=0; i< thread_num; ++i)
        {
            args[i].in = (void*)enclaveFile;
            args[i].i = i + 1;
            pthread_create(&threads[i], NULL, create_enclave, (void*)&(args[i]));
        }

        for(int i =0; i< thread_num; ++i)
        {
            pthread_join(threads[i], (void**)0);
        }
        printf("host: after exit the thread\n");
    out:
        elf_args_destroy(enclaveFile);
        free(enclaveFile);
        free(threads);
        free(args);

        return 0;
    }

It is a basic host app to manipulate an enclave. There are some host-side enclave-related APIs, such as ``PLenclave_create``, ``PLenclave_run``. etc.
You can find more detail in our :doc:`user manual <../Penglai-manual/User-Manual-TVM>`.

Penglai-TVM supports to run multiple enclaves in a single host. So, host can use the pthread to create multiple instances with single enclave file.

There are three basic structures to manipulate an enclave. ``struct elf_args``, ``struct PLenclave`` and ``struct enclave_args``.
``struct elf_args`` records the ELF file of given enclave. ``struct PLenclave`` is the primary enclave structure that host will use to create, run an enclave.
``struct enclave_args`` stores all enclave parameters, which will be used in enclave creation, running, etc.

You can use the following APIs to initialize the above structures.

.. code-block:: c 

  elf_args_init(enclaveFile, eappfile)

  PLenclave_init(enclave)
  
  enclave_args_init(params)

How to run an enclave
>>>>>>>>>>>>>>>>>>>>>>>>>
We provide several interfaces to manipulate an enclave.

.. code-block:: c

  PLenclave_create(enclave, enclaveFile, params)

We use this interface to create an enclave. Monitor will instantiate an enclave instance and return the corresponding ``eid``.
With the different parameters, host can create different kinds of enclaves: :doc:`shadow enclave <Tutorial-Penglai-TVM-shadow-enclave>`, :doc:`server enclave <Tutorial-Penglai-TVM-server-enclave>`, etc.

After the creation, enclave is not running, until someone invokes ``PLenclave_run``.

.. code-block:: c

  result = PLenclave_run(enclave)

Host invokes this API to run a created enclave. The enclave thread will not return to the host, 
unless: (1) Enclave has been finished and exited. (2) Enclave does an ocall and needs to return to user-level for handling.
If an enclave is finished successfully, the return result (not return value) remains zero.
The return value for enclave is stored in the return parameters: ``enclave->user_param.retval``

Enclave app
--------------

.. code-block:: c

    #include "eapp.h"
    #include <stdlib.h>

    int hello(unsigned long * args)
    {
        printf("hello world!\n");
        EAPP_RETURN(0);
    }

    int EAPP_ENTRY main(){
        unsigned long * args;
        EAPP_RESERVE_REG;
        hello(args);
    }

As for a very simple demo: hello-world running in enclave. It only needs a minor modification for the original program.
First, you need to include the enclave header file ``eapp.h`` in the enclave source file.
Second, you need to invoke the macro ``EAPP_RESERVE_REG`` to reserve the enclave context, just before jumping to the real enclave main function. 

We modify the musl lib to support the ``printf`` in enclave. So you can invoke the ``printf`` in the enclave directly.
This request will be redirected to the host kernel to handle. You can find more details on libc-supported APIs in the :doc:`user manual <../Penglai-manual/User-Manual-TVM>` for Penglai-TVM.



