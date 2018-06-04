# Universal MIG Parser
Extract and generate code based on name and type for mig func/arg/request&reply member etc, ideal helper for creating monitor, tracker, fuzzer etc for Mach Remote Procedure Calls.

[![Contact](https://img.shields.io/badge/contact-@cocoahuke-fbb52b.svg?style=flat)](https://twitter.com/cocoahuke) [![build](https://travis-ci.org/cocoahuke/universalmigparser.svg?branch=master)](https://travis-ci.org/cocoahuke/universalmigparser) [![license](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://github.com/cocoahuke/universalmigparser/blob/master/LICENSE) [![version](https://img.shields.io/badge/version-1.0-yellow.svg)]()


##### TL;DR [Click here to see generated code for a simple monitor](https://github.com/cocoahuke/universalmigparser/blob/master/Generated_code_sample.txt)

## The brief description of Mach RPC

**Mach RPC** is based on **Mach IPC** (Interprocess Communication), more explicit, is one use of the Mach Message. **Mach RPC** (Remote Procedure Call) can be generated by mig tool (Mach Interface Generator), has client/server system structure.  
In XNU kernel or another word, on MacOS system, a fair amount of MacOS-exclusive system calls used Mach RPC.

The well-know proc conception from BSD kernel, is associated with a ipc space object on Mach IPC, ipc space acts like a converter, converts ipc port object to a 32 bit integer which only meaningful in this ipc space, and ipc port object can store a kernel object, namely a kernel address point to a structure that contains rich members.
Overall it's similar to the concept of file descriptor from UNIX - Everything is a file. Mach message as a low-level primitive IPC mechanism, except UNIX system calls and Mach traps, almost all other IPC mechanisms are essentially using Mach Message, include Mach RPC.

XNU itself uses Mach RPC for enriches system calls, so it is also classified to many subsystems. (Files that end of `.defs` in XNU source code)

#### Mach RPC workflow
Mach RPC is actually passing a Mach message, and this is the structure contained in every Mach message:

```
typedef struct {
  mach_msg_bits_t	msgh_bits;
  mach_msg_size_t	msgh_size;
  mach_port_t		msgh_remote_port; aka msgh_request_port
  mach_port_t		msgh_local_port; aka msgh_reply_port
  mach_port_name_t	msgh_voucher_port;
  mach_msg_id_t		msgh_id;
} mach_msg_header_t;
```

Here I only explain some special places for Mach RPC, more about Mach message please read the relevant documents.

```
Call mach_msg at user-space -> msgh_remote_port |
                                                v
<-  msgh_local_port <- RPC server-side in kernel
```
`msgh_remote_port` is a ipc port, the ipc space which the port is associated will be the destination of the message delivery. This can explain such thing: You need to pass `masterport` for IOKit API functions that doesn't need any IO object as arguments, on the contrary, you don't need.

`msgh_local_port` is similar to `msgh_remote_port`, use when RPC server-side send back.

`msgh_id` will have a role after the message is delivered to the server side, technically can be used for any purpose, usually used as an index of function in Mach RPCs. Mach RPC who have the same server-side (Such as the kernel) requires a unique id, and `msgh_id` is where the value stored.

Then, entering kernel by mach trap (mach_msg or mach_msg_overwrite)

#### Mach RPC workflow - Kernel
I am not explaining the whole process. you can find more details from the book `Mac OS X and iOS: Internel To the Apple's Core`, at page 359 `Begind the Scenes of Message Passing`.

The following code explains the workflow of RPC from a certain perspective, although mach_msg_overwrite is for kernel use (mach_msg_overwrite_trap is for userspace, more stuff inside), it's more appropriate use as examples where (more clear).

Nevertheless, not all RPC server-side are kernel-resident, RPC server-side coul be part of a daemon process, and that will make little different scenario. such as `DiskArbitration <-> diskarbitrationd`. It's just like XPC, but higher efficiency.

###### mach_msg_overwrite internal implementation (code src: ipc_mig.c) :
```
if(option & MACH_SEND_MSG) ->
  ipc_kmsg_copyin ->
    -> ipc_kmsg_copyin_header
    if(msgh_bits & MACH_MSGH_BITS_COMPLEX) -> ipc_kmsg_copyin_body
    -> ipc_kmsg_send

if(option & MACH_RCV_MSG) ->
  ipc_mqueue_receive
  ipc_kmsg_copyout ->
    -> ipc_kmsg_copyout_header
    if(msgh_bits & MACH_MSGH_BITS_COMPLEX) -> ipc_kmsg_copyout_body

```

**Before** `ipc_kmsg_copyin`: Request port members are 32bit integer and only meaningful at sender's ipc space.  
**After** `ipc_kmsg_copyin`: Request port members replaced by the associated ipc object. (ipc object contain kernel object ptr)  
**Before** `ipc_kmsg_copyout`: RPC server-side completed execution, ready for send reply arguments back, port members are ipc object.  
**After** `ipc_kmsg_copyout`: Now the port member has been converted to 32-bit integer only meaningful at sender's ipc space.

You probably already noticed that the process of converting (`32bit integer <-> ipc port object`) is a good place for observe all passed Mach messages.

Yes, then you will need load a kext to hook these four functions, here is the code I use. It reminds me MSHookFunction(). [About how to program such kernel kext](aa).

```
/*
migtest_copyin and migtest_copyout is generated by MIG Parser.

Strong suggest putting the generated-code in a separate file,
because IDE took a long time for indexing every time I made modify.
*/
extern void migtest_copyin(ipc_kmsg_t kmsg, ipc_space_t space);
extern void migtest_copyout(ipc_kmsg_t kmsg, ipc_space_t space);

mach_msg_return_t (*orig_ipc_kmsg_copyin_header)(ipc_kmsg_t kmsg, ipc_space_t space, mach_msg_option_t *optionp);
mach_msg_return_t my_ipc_kmsg_copyin_header(ipc_kmsg_t kmsg, ipc_space_t space, mach_msg_option_t *optionp){

    mach_msg_return_t rt = orig_ipc_kmsg_copyin_header(kmsg, space, optionp);
    if(rt == MACH_MSG_SUCCESS && kmsg->ikm_header && ((kmsg->ikm_header->msgh_bits & MACH_MSGH_BITS_COMPLEX) != MACH_MSGH_BITS_COMPLEX)){
        migtest_copyin(kmsg, space);
    }
    return rt;
}

mach_msg_return_t (*orig_ipc_kmsg_copyin_body)(ipc_kmsg_t kmsg, ipc_space_t space, vm_map_t map);
mach_msg_return_t my_ipc_kmsg_copyin_body(ipc_kmsg_t kmsg, ipc_space_t space, vm_map_t map){

    mach_msg_return_t rt = orig_ipc_kmsg_copyin_body(kmsg, space, map);
    if(rt == MACH_MSG_SUCCESS){
      migtest_copyout(kmsg, space);
    }
    return rt;
}
...
```
Apply same way to `ipc_kmsg_copyout_head` and `ipc_kmsg_copyout_body`.

## So, what Universal MIG Parser can do
This project I brought to you was basically is a **Code Generator**, maximize possible customizations.  

#### Need input ?
You only need to specify an RPC user-side .c file.  [I will tell you how to get from xnu source.]()  
such as device_user.c, used to interactive with IOKit in the kernel.

Universal MIG Parser list all RPCs in the source file, and extract the following information from each function, AND turn them into code:

* Function Name
* Function Argument list
 * Type
 * Name
* Content in msgh_id
* Content in msgh_reply_port and msgh_request_port
* Request and Reply Structure member list
 * Type
 * Name
 * Array counting
 * Whether is function argument
 * Whether included by msgh_body  
* ...

"dynamic delta size" and "mig_reply_error_t potentially different formats reply data" problem is already solved for you.  
**Support all mig functions, your code can be integrated dynamically into hundreds of mig function in one go.**

Ideal helper for creating monitor, tracker, fuzzer for Mach RPCs.
Below is a monitor sample output (included in the source code), yes, read source code file and add or delete code by yourself.

It's the output style I designed, simple but extendable:  
`>` is request data.  
`<` is reply data.  
`X` is N/A.  
`(Type) Value` Print the corresponding code according to the variable type, you can add more always, refer to my code.

#### Example monitor sample output :
```
(SEND)io_server_version
  > mach_port_t master_port =
    (IPC Port Name) 0xb03
    (IPC Port Object) 0xffffff80208e02c0
  X uint64_t *version = (X)

(RECV)io_server_version
  X mach_port_t master_port = (X)
  < uint64_t *version =
    (uint64_t) 0x1335185

(SEND)io_registry_get_root_entry
  > mach_port_t master_port =
    (IPC Port Name) 0xb03
    (IPC Port Object) 0xffffff80208e02c0
  X mach_port_t *root = (X)

(RECV)io_registry_get_root_entry
  X mach_port_t master_port = (X)
  < mach_port_t *root =
    (IPC Port Name) 0xa0b
    (IPC Port Object) 0xffffff80262fff00

(SEND)io_registry_entry_create_iterator
  > mach_port_t registry_entry =
    (IPC Port Name) 0xa0b
    (IPC Port Object) 0xffffff80262fff00
  > io_name_t plane =
    (char*) IODeviceTree
  > uint32_t options =
    (uint32_t) 0x1
  X mach_port_t *iterator = (X)

(RECV)io_registry_entry_create_iterator
  X mach_port_t registry_entry = (X)
  X io_name_t plane = (X)
  X uint32_t options = (X)
  < mach_port_t *iterator =
    (IPC Port Name) 0xc03
    (IPC Port Object) 0xffffff803b3ce860

```
## No Makefile ?
As I said, you will need to modify the code to fit your need, often.  
Wrote in PURE C, nothing need to worry about.

## How to get RPC user-side source file from XNU source
It's a little tricky method but works:

##### Step 1
[Download](https://opensource.apple.com/tarballs/xnu/) the XNU source package.

##### Step 2
Search file ends up in `.defs`, then open `Makefile` beneath the same directory.

Example:  
`.defs` file path: `osfmk/device/device.defs`  
`Makefile` file path: `osfmk/device/Makefile`

Open `Makefile`, find below code:
```
${DEVICE_FILES}: device.defs
	@echo MIG $@
	$(_v)${MIG} ${MIGFLAGS} ${MIGKSFLAGS}	\
	-header /dev/null			\
	-user /dev/null				\
	-sheader device_server.h		\
	-server device_server.c			\
	$<
```
And replace to:
```
${DEVICE_FILES}: device.defs
	@echo MIG $@
	$(_v)${MIG} ${MIGFLAGS} ${MIGKSFLAGS}	\
	-header device_user.h			\
	-user device_user.c                	\
	-sheader device_server.h		\
	-server device_server.c			\
	$<
```
Save the changes. I believe you get it.

##### Step 3
`cd` to the root directory of xnu source package, start compiling.
```
make ARCH_CONFIGS="X86_64" KERNEL_CONFIGS=RELEASE SDKROOT=macosx
```
Probably will end up with error, even so, it does not affect, as long as you saw such line in the output: `MIG device_server.c`  
It appears in the very early stage during the compiling. And you should able to find RPC user-side source code file right next to the server-side file.

If you can't even generate server-side files, check your `dtrace` and `bootstrap_cmds` installation, they are necessary for start compiling.

## About program a kernel extension for function hooking

For now, you need figure out how to hook kernel function and write it down as kext, all by yourself. If you have done hook a C function by rewrite memory in the user-space before. you only need little extra work to make it work in the kernel.

Even after you successfully completed kext, you will encounter several problems more annoying:
 - Native kernel.framework is far from enough
 - Any changes need to reboot the whole system
 - Need include header files manually in order to have auto-completion in IDE
 - Undocumented function (Private KPI) or even no-symbol assembly
 - The Kext you've done cause the kernel panic on latest kernel version
 - ...

Hmmm, I may release other projects would help at least some of these problems.

Read interspersed comments in the code, sorry about lame English.  
[Click here to see generated code for a simple monitor](https://github.com/cocoahuke/universalmigparser/blob/master/Generated_code_sample.txt).
