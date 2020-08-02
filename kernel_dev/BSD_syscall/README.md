### BSD Syscalls - FreeBSD 12.1

### Structures

All the syscall structs and tables are defined at ***<sys/sysent.h>***, your syscall entry module should be defined using ***struct sysent***

```c
struct sysent {                 /* system call table */
        int     sy_narg;        /* number of arguments */
        sy_call_t *sy_call;     /* implementing function */
        au_event_t sy_auevent;  /* audit event associated with syscall */
        systrace_args_func_t sy_systrace_args_func;
                                /* optional argument conversion function. */
        u_int32_t sy_entry;     /* DTrace entry ID for systrace. */
        u_int32_t sy_return;    /* DTrace return ID for systrace. */
        u_int32_t sy_flags;     /* General flags for system calls. */
        u_int32_t sy_thrcnt;
};
```

The only important part here is the number of arguments and the function implementation address.

Example:

```c
static struct sysent syscall_sysent = {
	1,
	m_syscall
};
```

And to register our syscall inside our kernel, use ***SYSCALL_MODULE*** macro

Example:

```c
SYSCALL_MODULE(m_syscall, &sysc_offset, &syscall_sysent, load, NULL);
```

### Syscall tips

To find the syscall id using ***modfind*** there is some things to be aware, your syscall name will ***not be*** the first ***SYSCALL_MODULE*** parameter, take a look at this macro:

```c
#define SYSCALL_MODULE(name, offset, new_sysent, evh, arg)      \
static struct syscall_module_data name##_syscall_mod = {        \
        evh, arg, offset, new_sysent, { 0, NULL, AUE_NULL }     \
};                                                              \
                                                                \
static moduledata_t name##_mod = {                              \
        "sys/" #name,                                           \
        syscall_module_handler,                                 \
        &name##_syscall_mod                                     \
};                                                              \
DECLARE_MODULE(name, name##_mod, SI_SUB_SYSCALLS, SI_ORDER_MIDDLE)
```

Notice that SYSCALL_MODULE it's only a macro to DECLARE_MODULE, but before call DECLARE_MODULE, our module name is changed to ***"sys/" #name***, so my syscall above, m_syscall, will have the module name ***sys/m_syscall***


### Calling from C

```c
#include <stdio.h>
#include <string.h>
#include <errno.h>

#include <sys/syscall.h>
#include <sys/types.h>
#include <sys/module.h>
...

    int mod_id, syscall_num;
	struct module_stat stat;
	stat.version = sizeof(stat);
	
	mod_id = modfind("sys/m_syscall");
    modstat(mod_id, &stat);

	syscall_num = stat.data.intval;
	syscall(syscall_num, /* arguments */);
```

### Calling from Python

```python
>>> import ctypes
>>> libc = ctypes.CDLL(None)
>>> syscall = libc.syscall
>>> syscall(210, "Something nice here".encode())
0
```

### Calling from Ruby

```ruby
require 'ffi'

module SyscallRunner
  extend FFI::Library
  ffi_lib FFI::Library::LIBC
  attach_function :syscall, [:int, :int], :int
end

SyscallRunner.syscall 60, 1
```

You can track the logs from ***dmesg*** output