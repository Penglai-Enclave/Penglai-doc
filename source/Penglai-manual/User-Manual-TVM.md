# User manual for Penglai-TVM

### Introduction

This doc introduces how to create or customize an enclave in the Penglai-TVM, such as how to configure the enclave parameters in Penglai sdk and what's the usage of each enclave-side / host-side interface, etc.

Penglai-TVM is a scalable enclave system on RISC-V, which realizes the fine-grained and scalable memory management. We leverage the RISC-V feature: Trap Virtual Memory (TVM) to implement the Guard Page Table with pure software design. More details of Guard Page Table can be found in our [OSDI paper](https://ipads.se.sjtu.edu.cn/zh/publications/FengOSDI21-preprint.pdf).

### File Structure

```shell
# Penglai TVM version, a pure software enclave design on risc-v.
.
├── conf                  // The configuration for Linux and Buildroot
├── copy-files            // Copy the files into ramfs
├── docker_cmd.sh         // Docker command file
├── docs                  // Docs for Penglai  
├── LICENSE               // License for Penglai
├── Makefile              // Makefile  
├── penglai-buildroot     // Penglai buildroot
├── Penglai-Linux-TVM     // Penglai Linux kernel (5.10.2)
├── Penglai-Opensbi-TVM   // Penglai Opensbi, including secure monitor and crypto lib
├── penglai-qemu          // RISC-V QEMU suppoet sPMP extension
├── Penglai-sdk-TVM       // Penglai sdk, demo and kernel driver 
├── README.md             // Penglai README 
├── scripts				        // Scripts for building , running qemu	
└── work                  // Build target file, including linux kernel and buildroot 
```

There are three key submodules in Penglai: Linux (with Guarded Page Table support), sdk (host and enclave library and enclave driver) and Opensbi (including secure monitor). 

### Host-side interface

#### Enclave management interfaces

+ **PLenclave_init**

  ```c
  int PLenclave_init(struct PLenclave *PLenclave)
  ```

  + Description: This function is used to initialize the enclave metadata.
  + Parameter:
    + PLenclave: enclave structure used in the host program.

+ **PLenclave_create**

  ```c
  int PLenclave_create(struct PLenclave* PLenclave, struct elf_args* u_elffile, struct enclave_args* u_param)
  ```

  + Description: Create an enclave with the given parameters and enclave file.
  + Parameter:
    + PLenclave: enclave structure used in the host program.
    + u_elffile: the elf file structure of the created enclave.
    + u_param: user-given parameters to create an enclave.

+ **PLenclave_run**

  ```c
  int PLenclave_run(struct PLenclave *PLenclave)
  ```

  + Description: Run an enclave, this function will not return unless (1) enclave is finished, stopped or destroyed, (2) enclave triggers an ocall which needs to be handled in the user mode (host).
  + Parameter:
    + PLenclave: enclave structure used in the host program.

+ **PLenclave_attest**

  ```c
  int PLenclave_attest(struct PLenclave *PLenclave, uintptr_t nonce)
  ```

  + Description: Get the attestation report according to the given nonce
  + Parameter:
    + PLenclave: enclave structure used in the host program.
    + nonce: attestation nonce.

+ **PLenclave_stop**

  ```c
  int PLenclave_stop(struct PLenclave *PLenclave)
  ```

  + Description: Stop a running enclave. The host can stop a running enclave compulsorily, So the DoS attacks from the untrusted host are out of scope.
  + Parameter:
    + PLenclave: enclave structure used in the host program.

+ **PLenclave_resume**

  ```c
  int PLenclave_resume(struct PLenclave *PLenclave)
  ```

  + Description: Resume a stopped enclave. An enclave can continue to run from the point it exits last time.
  + Parameter:
    + PLenclave: enclave structure used in the host program.
  
+ **PLenclave_destroy**

  ```c
  int PLenclave_destroy(struct PLenclave *PLenclave)
  ```

  + Description: Destroy an enclave. The host can destroy a running enclave compulsorily, e.g., when receiving a kill signal. DoS attacks are out of scope.
  + Parameter:
    + PLenclave: enclave structure used in the host program.

#### Enclave paremeter interfaces

##### Running parameter

**elf_args_init**

```c
void elf_args_init(struct elf_args* elf_args, char *filename)
```

+ Description: Initialize the elf structure with the given filename.
+ Parameter:
  + elf_args: elf structure used in the host program.
  + filename: enclave file name.

**elf_args_destroy**

```c
void elf_args_destroy(struct elf_args* elf_args)
```

+ Description: Reclaim the elf resource.
+ Parameter:
  + elf_args: elf structure used in the host program.

**enclave_args_init**

```c
void enclave_args_init(struct enclave_args* enclave_args)
```

+ Description: Initialize the enclave arguments.
+ Parameter:
  + enclave_args: enclave argument structure used in the host program.

**PLenclave_set_shm**

```c
int PLenclave_set_shm(struct PLenclave *enclave, int shmid, uintptr_t offset, uintptr_t size)
```

+ Description: Configure the shared memory in the enclave creating parameters.
+ Parameter:
  + enclave: enclave structure used in the host program.
  + shmid: shared memory identification.
  + offset: offset in shared memory.
  + size: shared memory size.

**PLenclave_set_mem_arg**

```c
int PLenclave_set_mem_arg(struct PLenclave *enclave, int id, uintptr_t offset, uintptr_t size)
```

+ Description: Set the schrodinger pages in the enclave running parameters. The ownership of schrodinger pages is transferred when enclave runs.
+ Parameter:
  + enclave: enclave structure used in the host program.
  + id: schrodinger page identification.
  + offset: offset in schrodinger pages.
  + size: schrodinger page size.

**PLenclave_set_rerun_arg**

```c
int PLenclave_set_rerun_arg(struct PLenclave *enclave, int rerun_reason)
```

+ Description: Set the enclave re-run cause in its parameter.
+ Parameter:
  + enclave: enclave structure used in the host program.
  + rerun_reason: the cause for enclave to re-run.

##### Shared memory

**PLenclave_shmget**

```c
int PLenclave_shmget(unsigned long size);
```

+ Description: Allocate the shared memory between the enclave and host, and return the shared memory identification.
+ Parameter:
  + size: shared memory size.
+ Return value: 
  + shmid: the identification of the required shared memory

**PLenclave_shmat**

```c
void* PLenclave_shmat(int shmid, void* addr)
```

+ Description: Get the shared memory address with the given shmid. Map this shared memory into host VA space.
+ Parameter:
  + shmid: shared memory identification.
  + addr: shared memory address. 0 (default) means shared memory can map at any available virtual address in the VA space of the host.
+ Return value: 
  + addr: the virtual address of the shared memory.

**PLenclave_shmdt**

```c
int PLenclave_shmdt(int shmid, void* addr)
```

+ Description: Unmap the shared memory in the host VA space with the given shmid. However, the shared memory id is not deallocated, so others can still use this shmid to get the corresponidng shared memory.
+ Parameter:
  + shmid: shared memory identification.
  + addr: shared memory address.

**PLenclave_shmctl**

```c
int PLenclave_shmctl(int shmid)
```

+ Description: Reclaim the shared memory identification. The host cannot use this shmid to get the corresponding shared memory any longer.
+ Parameter:
  + shmid: shared memory identification.

##### Schrodinger/Relay page (Zero-copy mechanism)

**PLenclave_schrodinger_get**

```c
int PLenclave_schrodinger_get(unsigned long size)
```

+ Description: Allocate the schrodinger page between the enclave and host (achieve the zero copy communication), and return its identification.
+ Parameter:
  + size: schrodinger page size.
+ Return value: 
  + schrodinger_id: schrodinger page identification.

**PLenclave_schrodinger_at**

```c
void* PLenclave_schrodinger_at(int id, void* addr)
```

+ Description: Get the schrodinger page address with the given id. Map these schrodinger pages into host VA space.
+ Parameter:
  + id: schrodinger page identification.
  + addr: schrodinger page address. 0 means the schrodinger page can map at any available virtual address in the VA space of the host.
+ Return value: 
  + addr: the virtual address of the schrodinger pages.

**PLenclave_schrodinger_dt**

```c
int PLenclave_schrodinger_dt(int id, void* addr)
```

+ Description: Unmap the schrodinger pages in the host VA space with the given shmid. However, the schrodinger page id is not deallocated, so others can still use this id to get the corresponidng schrodinger pages.
+ Parameter:
  + id: schrodinger page identification.
  + addr: schrodinger page address.

**PLenclave_schrodinger_ctl**

```c
int PLenclave_schrodinger_ctl(int id)
```

+ Description: Reclaim the schrodinger page identification. The host cannot use this id to get the corresponding schrodinger pages any longer.
+ Parameter:
  + id: schrodinger page identification.

### Enclave-side interface

#### Libc supported

We have integrated the `musl libc`into the enclave-side library. It can support several unmodified libc interfaces:

+ Print
  + printf
+ Memory related:
  + malloc
  + calloc
  + free
  + sbrk
+ FS related:
  + fopen
  + fputs
  + fgets
  + fclose
  + stat

To support the FS-related interfaces, it needs to run a fs server in the enclave. We provide `lfs` as the fs server, and can run successfully in the penglai enclave.

Other interfaces like `memset()`,  `memcpy()`, etc, have no interaction with kernel, which are also supported in Penglai enclave. 

#### Enclave-specific interfaces

**EAPP_RETURN**

```c
void EAPP_RETURN(unsigned long retval) __attribute__((noreturn))
```

+ Description: Exit an enclave and give the return value.
+ Parameter:
  + retval: return value.

**EAPP_GET_ENCLAVE_ID**

```c
unsigned long get_enclave_id()
```

+ Description: Get the current enclave identification.
+ Return value: 
  + eid: enclave id.

**EAPP_MMAP**

```c
void* eapp_mmap(void* vaddr, unsigned long size)
```

+ Description: Allocate the enclave memory and map it in the enclave VA space. These memory can be used as the relay pages in enclave (zero-copy communication between enclaves).
+ Parameter:
  + vaddr: set NULL in the current version.
  + size: memory mapping size 
+ Return value: 
  + addr: the virtual address for mapped memory.

**EAPP_UNMAP**

```c
int eapp_unmap(void* vaddr, unsigned long size)
```

+ Description: Unmap the enclave memory. The unmapped memory must be allocated with the `eapp_mmap()`.
+ Parameter:
  + vaddr: the virtual address of memory which is going to unmap.
  + size: unmapped memory size.

**EAPP_ACQUIRE_ENCLAVE**

```c
unsigned long acquire_enclave(char* name)
```

+ Description: Acquire the server enclave handler with the given enclave name. The enclave handler can be used in enclave call.
+ Parameter:
  + name: enclave name.
+ Return value: 
  + handler: enclave handler.

**EAPP_CALL_ENCLAVE**

```c
int call_enclave(unsigned long handle, struct call_enclave_arg_t* arg)
```

+ Description: Synchronous enclave call. A caller enclave can use this interface to call other enclaves. Caller enclave must wait until the callee enclave returns.
+ Parameter:
  + handle: callee enclave handler.
  + arg: IPC structure.

**EAPP_ASYN_ENCLAVE_CALL**

```c
int asyn_enclave_call(char* name, struct call_enclave_arg_t *arg)
```

+ Description: Asynchronous enclave call. A caller enclave can use this interface to pass the IPC arguments to the callee enclave. The caller enclave will continue to run and callee enclave will receive the IPC arguments when it creates.
+ Parameter:
  + name: callee enclave name.
  + arg: IPC structure.

**SERVER_RETURN**

```c
void SERVER_RETURN(struct call_enclave_arg_t *arg) __attribute__((noreturn))
```

+ Description: Callee enclave calls this function to return back to the caller enclave, and transfers the return IPC structure in the meantime.
+ Parameter:
  + arg: IPC structure.

**EAPP_ENCLAVE_REPORT**

```c
int EAPP_GET_REPORT(char * name, struct report_t *report, unsigned long nonce)
```

+ Description: Get the enclave attestation report. If name is set, it will return the enclave report with the given name, otherwise, it will return the attestation report of the current enclave. 
+ Parameter:
  + name: attested enclave name.
  + report: attestation report structure, the monitor will fill this structure later.
  + nonce: attestation nonce.

### Configuration

+ **Enclave memory layout:**

  ```c
  /* default layout of enclave */
  /*
  #####################
  #   reserved for    #
  #       s mode      #
  ##################### 0xffffffe000000000 //the start address of kernel's image
  #       hole        #
  ##################### 0x0000004000000000
  #    shared mem     #
  #     with host     #
  ##################### 0x0000003900000000
  #                   #
  #    host mm arg    #
  #                   #
  ##################### 0x0000003800000000
  #                   #
  #       stack       #
  #                   #
  ##################### 0x0000003000000000
  #       mmap        #
  #                   #
  ##################### brk
  #                   #
  #       heap        #
  #                   #
  ##################### 0x0000001000000000
  #                   #
  #   text/code/bss   #
  #                   #
  ##################### 0x0000000000001000 //not fixed, depends on enclave's lds
  #       hole        #
  ##################### 0x0
  */
  ```

  Enclave memory layout is defined in the `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_STACK_SIZE**

  + Description: 
    + Default enclave stack size: 8K
  + Defined file:
    + `/Penglai-sdk-TVM/lib/host/include/param.h`
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_STACK_BASE**

  + Description:
    + Default enclave stack base address: 0x0000003800000000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_KBUFFER**

  + Description:
    + The start address of shared memory between kernel and enclave: 0xffffffe000000000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`
    + `/Penglai-sdk-TVM/lib/app/include/ocall.h`
    + `/Penglai-sdk-TVM/enclave-driver/penglai-enclave.h`

+ **ENCLAVE_DEFAULT_KBUFFER_SIZE**

  + Description:
    + The default size of the shared memory between the kernel and enclave: 1K
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`
    + `/Penglai-sdk-TVM/lib/app/include/ocall.h`
    + `/Penglai-sdk-TVM/enclave-driver/penglai-enclave.h`

+ **ENCLAVE_DEFAULT_SHM_BASE**

  + Description:
    + Shared memory between host and enclave, the default start address in the enclave VA space: 0x0000003900000000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_MM_ARG_BASE**

  + Description:
    + Zero-copy memory start address in the enclave VA space: 0x0000003900000000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_MMAP_BASE**

  + Description:
    + Mmap memory range start address in the enclave VA space: 0x0000003000000000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_HEAP_BASE**

  + Description:
    + Heap start address in the enclave VA space: 0x0000001000000000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`

+ **ENCLAVE_DEFAULT_TEXT_BASE**

  + Description:
    + Text section start address  (entry point) in the enclave VA space: 0x0000000000001000UL
  + Defined file:
    + `/Penglai-Opensbi-TVM/include/sm/enclave_vm.h`
    + `/Penglai-sdk-TVM/app.lds`

+ **DEFAULT_SHADOW_ENCLAVE_ORDER**

  + Description:
    + The initialized page order of shadow enclave. These pages will be used to create a shadow enclave instance (used as heap and stack memory) later
  + Defined file:
    + `/Penglai-sdk-TVM/enclave-driver/penglai-enclave.h`

+ **DEFAULT_SECURE_PAGES_ORDER**

  + Description:
    + The page order of secure memory. Penglai driver transfers a range of memory into monitor (used as secure memory), when secure memory held by monitor is exhausted.
  + Defined file:
    + `/Penglai-sdk-TVM/enclave-driver/penglai-enclave.h`
  
+ **DEFAULT_SCHRODINGER_ORDER**

  + Description:
    + The page order of zero-copy communication pages. Penglai driver allocates all the schrodinger pages when it initializes. These pages will be assigned to each enclave if needed (as its schrodinger pages).  
  + Defined file:
    + `/Penglai-sdk-TVM/enclave-driver/penglai-schrodinger.h`



