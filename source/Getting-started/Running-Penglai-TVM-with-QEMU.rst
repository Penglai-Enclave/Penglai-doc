Running Penglai-TVM with QEMU
==========================

Environment requirement
------------------------
Penglai uses Docker for building and uses submodules to track different componets.

Therefore, the only requirement to build and run penglai-demo is:

+ `Docker <https://docs.docker.com/>`_: for building/running Penglai
+ Git: for downloading the source code
 
Get source code
----------------
The source code of Penglai-TVM is available, click `here <https://github.com/Penglai-Enclave/Penglai-Enclave-TVM>`_ to jump to our git repo.
You can clone our project with the following script.
::
  git clone https://github.com/Penglai-Enclave/Penglai-Enclave-TVM.git

Building
---------
Enter the penglai-enclave directory, cd ``Penglai-Enclave-TVM``

And then,
:: 
  git submodule update --init --recursive

Last, build penglai using our Docker image:
::
  ./docker_cmd.sh build

When the building process finished, you are ready to run penglai.

.. warning::
             In current verrsion, building maybe failed but will not stop the build procedure. So you need to check the build output and make sure there is no error.

Running
--------
Enter the ``Penglai-Enclave-TVM`` directory

Typing,
:: 
  ./docker_cmd qemu

If everything is fine, you will enter a Linux terminal booted by Qemu with Penglai-installed.

Enter the terminal with the user name: ``root``, and passwords: ``penglai``.

**Insmod the enclave-driver**
:: 
   sh install.sh

And then, you can run a demo, e.g., a hello-world enclave, using
::
   ./host hello-world
  
Here, the host is an enclave invoker, which will start an enclave (name from input).

  
  
