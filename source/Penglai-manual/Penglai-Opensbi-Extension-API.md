# Penglai opensbi extension API

### Introduction

This document describes the interfaces for Opensbi used by PENGLAI enclave. It contains two parts: one is called by the host kernel (Host-side SBI extension); Another is invoked by enclaves (Enclave-side SBI extension).

### Host-side SBI extension for  Penglai enclave

+ **Extension id: 0x100100**

+ **Common interfaces for Penglai Enclave:**

  The common interfaces must be implemented in Penglai monitor.

  | Macro               | Function ID | Argument                                                     |
  | ------------------- | ----------- | ------------------------------------------------------------ |
  | SBI_CREATE_ENCLAVE  | 99          | **arg0**: enclave create parameters                          |
  | SBI_ATTEST_ENCLAVE  | 98          | **arg0**: eid<br />**arg1**: the return report of enclave measurement<br />**arg2**: nonce |
  | SBI_RUN_ENCLAVE     | 97          | **arg0**: eid<br />**arg1**: enclave run parameters          |
  | SBI_STOP_ENCLAVE    | 96          | **arg0**: eid                                                |
  | SBI_RESUME_ENCLAVE  | 95          | **arg0**: eid<br />**arg1**: resume_func_id                  |
  | SBI_DESTROY_ENCLAVE | 94          | **arg0**: eid                                                |

+ **Functions**

  **SBI_CREATE_ENCLAVE （FID #99）**

  ```c
  typedef struct enclave_create_param
  {
    unsigned int *eid_ptr;
    char name[NAME_LEN];
    enclave_type_t type;
  
    unsigned long paddr;
    unsigned long size;
  
    unsigned long entry_point;
  
    unsigned long free_mem;
  
    //enclave shared mem with kernel
    unsigned long kbuffer;//paddr
    unsigned long kbuffer_size;
  
    //enclave shared mem with host
    unsigned long shm_paddr;
    unsigned long shm_size;
  
    unsigned long *ecall_arg0;
    unsigned long *ecall_arg1;
    unsigned long *ecall_arg2;
    unsigned long *ecall_arg3;
    unsigned long *retval;
    //extended params
    //...
  }
  
  // enclave_create_param->a0
  uintptr_t sbi_create_enclave(struct enclave_create_param *enclave_create_param)
  ```

  Create an enclave with the given arguments, monitor will initialize the enclave metadata, but will not run this enclave. 

  + Arguments:
    + enclave_create_param: struct of the enclave creating parameters.

  **SBI_ATTEST_ENCLAVE （FID #98）**

  Get the attestation report with the given nonce and eid. User can verify the attestation report later.

  ```c
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
  
  // eid->a0, report->a1, nonce->a2
  uintptr_t sbi_attest_enclave(unsigned long eid, struct report_t *report, unsigned long nonce);
  ```

  + Arguments:
    + eid: attested enclave id
    + report: struct pointer of attestation report, host will receive this report after attestation.
    + nonce: attestation nonce

  **SBI_RUN_ENCLAVE （FID #97）**

  Run an enclave which is already created. Monitor will reserve the running context of the host (regs and ocall handler, etc.), and switch to the enclave mode.

  ```c
    struct enclave_run_param
    {
      //zero-copy memory range, optional!
      unsigned long mm_arg_addr;
      unsigned long mm_arg_size;
      //extended paramseters
      //...
    } ;
    
    // eid->a0, enclave_run_arg->a1
    uintptr_t sbi_run_enclave(unsigned long eid, struct enclave_run_param * enclave_run_arg);
  ```

  + Arguments:
    + eid: given enclave id
    + enclave_run_args: struct of enclave running parameters

  **SBI_STOP_ENCLAVE （FID #96）**

  Host kernel can stop a running enclave compulsorily. A stopped enclave can resume later, so monitor needs to save the enclave context and cannot free enclave resource . DoS attack is out of scope.

  ```c
    // eid->a0
    uintptr_t sbi_stop_enclave(unsigned long eid);
  ```

  + Arguments:
    + eid: given enclave id 

  **SBI_RESUME_ENCLAVE （FID #95）**

  Re-run an enclave. An enclave can be stopped or be scheduled out (e.g., invoke ocall function). However, the host can use this function to resume an enclave with its previous context.

  ```c
  #define RESUME_FROM_TIMER_IRQ    0
  #define RESUME_FROM_STOP         1
  #define RESUME_FROM_OCALL        2  
  
  // eid->a0, resume_func_id->a1
    uintptr_t sbi_resume_enclave(unsigned long eid, unsigned long resume_func_id);
  ```

  + Arguments:
    + eid: the given enclave id
    + resume_func_id: the resume reason for a stopped enclave
      + RESUME_FROM_TIMER_IRQ  0 
      + RESUME_FROM_STOP            1
      + RESUME_FROM_OCALL          2  

  **SBI_DESTROY_ENCLAVE （FID #94）**

  Host can destroy an enclave (e.g., kill the host program will also kill the corresponding enclave). Monitor will clear all the enclave metadata and free its resource. Dos attack is out of the scope.

  ```c
    // eid->a0
    uintptr_t sbi_destroy_enclave(unsigned long eid)
  ```

  + Arguments
    + eid: given enclave id.

  

+ **SBI return value:**

  + sbi_error:

    | `a0`                                | Error number | Description                             |
    | ----------------------------------- | ------------ | --------------------------------------- |
    | SBI_ENCLAVE_UNDEFINED_ERROR         | -1           | Enclave undefined error                 |
    | SBI_ENCLAVE_NO_MEM                  | -2           | Monitor has no memory to create enclave |
    | SBI_ENCLAVE_NOT_EXISTED             | -3           | Enclave is not existed                  |
    | SBI_ENCLAVE_ALREADY_EXISTED         | -4           | Enclave is already existed              |
    | SBI_ENCLAVE_ALLOC_FAILED            | -5           | Allocating an enclave is failed         |
    | SBI_ENCLAVE_ARG_ERROR               | -6           | Error arguments                         |
    | SBI_ENCLAVE_ATTESTATION_FAILED      | -7           | Enclave attestation is failed           |
    | SBI_ENCLAVE_SECURITY_CHECK_FAILED   | -8           | Enclave security check is failed        |
    | SBI_ENCLAVE_IPI_NOTIFICATION_FAILED | -9           | Enclave ipi notification is failed      |
    
  + sbi return val:

    | `a1`             | val  |
    | ---------------- | ---- |
    | ENCLAVE_SUCCESS  | 0    |
    | ENCLAVE_OCALL    | 1    |
    | ENCLAVE_TIME_IRQ | 2    |


### Enclave-side SBI extension for  Penglai enclave

+ **S+U enclave**

  + Extension id: 0x100101
  + Use the sbi trap of CAUSE_SUPERVISOR_ECALL.
  
+ **U enclave**

  + Extension id: 0x100101
  + Use the sbi trap of CAUSE_USER_ECALL, which is undefined in the sbi specification.

+ **Functions:**

  | Macro               | Function ID (a7) | Argument                                                     |
  | ------------------- | ---------------- | ------------------------------------------------------------ |
  | SBI_EXIT_ENCLAVE    | 99                | **arg0**: return value                                       |
  | SBI_ENCLAVE_OCALL   | 98                | **arg0**: ocall_id<br />**arg1...**: the ocall arguments     |
  | SBI_ACQUIRE_ENCLAVE | 97                | **arg0**: The acquired enclave name                          |
  | SBI_CALL_ENCLAVE    | 96                | **arg0**: The callee enclave id<br />**arg1**: IPC struct    |
  | SBI_ENCLAVE_RETURN  | 95                | **arg0**: IPC struct                                         |
  | SBI_GET_REPORT      | 94               | **arg0**: the attested enclave name<br />**arg1**: attestation report<br />**arg2**: nonce |

  **SBI_EXIT_ENCLAVE （FID #1）**

  Enclave exits and returns to the host. Monitor clears the enclave metadata and frees all its resource.

  ```c
  // retval->a0
  uintptr_t exit_enclave(unsigned long retval)
  ```

  + Arguments:
    + retval: enclave return value.

  **SBI_ENCLAVE_OCALL （FID #2）**

  Enclave ocall functions, enclave uses ocall to invoke the functionalities in host kernel. Monitor will switch CPU status from enclave to host.

  ```c
  // ocall_id->a0, arg0->a1, arg1->a2 
  // return value: ENCLAVE_OCALL, the reason of enclave to exit
  uintptr_t enclave_ocall(unsigned long ocall_id, unsigned long arg0, unsigned long arg1)
  ```

  + Arguments:
    + ocall_id: ocall function id.
    + arg0...: ocall call arguments.
  + Return value:
    + ocall_return_value: the return value of ocall function (non-return value is also allowed).

  **SBI_ACQUIRE_ENCLAVE （FID #3）**

  A caller enclave uses this interface to get the handler of other enclaves with their enclave name. The enclave handler can be used to call another enclave. Monitor already reserves each server enclave name (when server enclave creates) and ensures that each enclave name is unique.

  ```c
  // acquired_name->a0 
  // return value: acquired enclave id 
  uintptr_t server_enclave_acquire(uintptr_t enclave_name)
  ```

  + Arguments

    + enclave_name: acquired enclave name
  + Return value:
    + handler : acquired enclave handler (enclave identification as well).

  **SBI_CALL_ENCLAVE （FID #4）**

  After getting the callee enclave handler with the given name. A caller enclave can trigger an inter-enclave communication between caller and callee enclave, which is defined as synchronous enclave call. Monitor can adopt different IPC mechanisms, and switch the running context from caller enclave to the callee enclave directly.

  ```c
  struct call_enclave_arg
  {
    uintptr_t req_arg;
    uintptr_t resp_val;
    uintptr_t req_vaddr;
    uintptr_t req_size;
    uintptr_t resp_vaddr;
    uintptr_t resp_size;
    //extended paramseters
    //...
  };
  
  //eid->a0, call_arg->a1
  uintptr_t sm_call_enclave(unsigned long eid, struct call_enclave_arg *call_arg)
  ```

  +  Arguments:
    + eid: callee enclave id
    + call_arg: IPC struct which contains the register and memory ranges

  **SBI_ENCLAVE_RETURN （FID #5）**

  When callee enclave exits and returns to the caller enclave, it needs to invoke this function. Monitor will switch running context from callee enclave to the caller enclave, and give back the resource owned by the caller enclave preciously.

  ```c
  struct call_enclave_arg
  {
    uintptr_t req_arg;
    uintptr_t resp_val;
    uintptr_t req_vaddr;
    uintptr_t req_size;
    uintptr_t resp_vaddr;
    uintptr_t resp_size;
    //extended paramseters
    //...
  };
  
  // return_arg->a0
  uintptr_t sm_enclave_return(struct call_enclave_arg *return_arg)
  ```

  + Arguments:
    + return_arg: return IPC struct which contains register and memory range

  **SBI_GET_REPORT（FID #11）**

  An enclave can invoke this function to get the attestation report. If enclave name is not given, it will return the attestation report of itself, otherwise, it will retrieve the report of the attested enclave corresponding to the given enclave name. This function can be used in the local/remote attestation.

  ```c
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
  
  // return_arg->a0
  uintptr_t sm_get_report(char *name, struct report_t *report, unsigned long nonce);
  ```

  + Arguments:
    + name: attested enclave name, NULL for current enclave itself.
    + report: attestation report structure, after invocation, the attestation report will be filled into this structure.
    + nonce: the nonce of attestation request.

+ **SBI return value:**
  
  Currently, the return value for enclave-side SBI call resides in the `a0`. We do not use the standard SBI return structure (`a0` for SBI_ERROR and `a1` for SBI return value), as we need to support the return value for a syscall.
  
  If enclave-side SBI call is failed, the return value is a negative error number. If an enclave-side SBI call does not have a return value and return successfully, the return value will be set to zero.

  | `a0`            | val  |
  | --------------- | ---- |
  | ENCLAVE_SUCCESS | 0    |
  

  
  