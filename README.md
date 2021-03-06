# async_wake
> iOS 11.1.2 kernel exploit and PoC local kernel debugger by @i41nbeer

___

## Supported Devices:
 tfp0: all 64-bit devices running 11.1.2

 tfp0 + local kernel debugger: the devices I have on my desk running 11.1.2 (iPhone 7, iPhone 6s, iPod Touch 6G)
                               theoretically it will also work for all other devices, you just need to find the symbols

## PoC local kernel debugger:
You can pause the execution of a syscall at arbitrary points and modify kernel state (registers and memory) then continue it.
See kdbg.c for details and implementation.

## The bugs:

=== CVE-2017-13861 ===
[https://bugs.chromium.org/p/project-zero/issues/detail?id=1417]

I have previously detailed the lifetime management paradigms in MIG in the writeups for:
CVE-2016-7612 [https://bugs.chromium.org/p/project-zero/issues/detail?id=926]
and
CVE-2016-7633 [https://bugs.chromium.org/p/project-zero/issues/detail?id=954]

If a MIG method returns KERN_SUCCESS it means that the method took ownership of *all* the arguments passed to it.
If a MIG method returns an error code, then it took ownership of *none* of the arguments passed to it.

If an IOKit userclient external method takes an async wake mach port argument then the lifetime of the reference
on that mach port passed to the external method will be managed by MIG semantics. If the external method returns
an error then MIG will assume that the reference was not consumed by the external method and as such the MIG
generated coode will drop a reference on the port.

IOSurfaceRootUserClient external method 17 (s_set_surface_notify) will drop a reference on the wake_port
(via IOUserClient::releaseAsyncReference64) then return an error code if the client has previously registered
a port with the same callback function.

The external method's error return value propagates via the return value of is_io_connect_async_method back to the
MIG generated code which will drop a futher reference on the wake_port when only one was taken.

I also use another bug:

=== CVE-2017-13865 ===
[https://bugs.chromium.org/p/project-zero/issues/detail?id=1372]
the kernel libproc API proc_list_uptrs has the following comment in it's userspace header:

/*
 * Enumerate potential userspace pointers embedded in kernel data structures.
 * Currently inspects kqueues only.
 *
 * NOTE: returned "pointers" are opaque user-supplied values and thus not
 * guaranteed to address valid objects or be pointers at all.
 *
 * Returns the number of pointers found (which may exceed buffersize), or -1 on
 * failure and errno set appropriately.
 */

This is a recent addition to the kernel, presumably as a debugging tool to help enumerate
places where the kernel is accidentally disclosing pointers to userspace.

The implementation currently enumerates kqueues and dumps a bunch of values from them.

Here's the relevant code:

// buffer and buffersize are attacker controlled

int
proc_pidlistuptrs(proc_t p, user_addr_t buffer, uint32_t buffersize, int32_t *retval)
{
  uint32_t count = 0;
  int error = 0;
  void *kbuf = NULL;
  int32_t nuptrs = 0;

  if (buffer != USER_ADDR_NULL) {
    count = buffersize / sizeof(uint64_t);     <---(a)
    if (count > MAX_UPTRS) {
      count = MAX_UPTRS;
      buffersize = count * sizeof(uint64_t);
    }
    if (count > 0) {
      kbuf = kalloc(buffersize);               <--- (b)
      assert(kbuf != NULL);
    }
  } else {
    buffersize = 0;
  }

  nuptrs = kevent_proc_copy_uptrs(p, kbuf, buffersize);

  if (kbuf) {
    size_t copysize;
    if (os_mul_overflow(nuptrs, sizeof(uint64_t), &copysize)) {  <--- (c)
      error = ERANGE;
      goto out;
    }
    if (copysize > buffersize) {    <-- (d)
      copysize = buffersize;
    }
    error = copyout(kbuf, buffer, copysize);  <--- (e)
  }


At (a) the attacker-supplied buffersize is divided by 8 to compute the maximum number of uint64_t's
which can fit in there.

If that value isn't huge then the attacker-supplied buffersize is used to kalloc the kbuf buffer at (b).

kbuf and buffersize are then passed to kevent_proc_copy_uptrs. Looking at the implementation of
kevent_proc_copy_uptrs the return value is the total number of values it found, even if that value is larger
than the supplied buffer. If it finds more than will fit it keeps counting but no longer writes them to the kbuf.

This means that at (c) the computed copysize value doesn't reflect how many values were actually written to kbuf
but how many *could* have been written had the buffer been big enough.

If there were possible values which could have been written than there was space in the buffer then at (d) copysize
will be limited down to buffersize.

Copysize is then used at (e) to copy the contents of kbuf to userspace.

The bug is that there's no enforcement that (buffersize % 8) == 0. If we were to pass a buffersize of 15, at (a) count would be 1
as 15 bytes is only enough to store 1 complete uint64_t. At (b) this would kalloc a buffer of 15 bytes.

If the target pid actually had 10 possible values which kevent_proc_copy_uptrs finds then nuptrs will return 10 but it will
only write to the first value to kbuf, leaving the last 7 bytes untouched.

At (c) copysize will be computed at 10*8 = 80 bytes, at (d) since 80 > 15 copysize will be truncated back down to buffersize (15)
and at (e) 15 bytes will be copied back to userspace even though only 8 were written to.


## Exploit technique:
I use the proc_pidlistuptrs bug to disclose the address of arbitrary ipc_ports. This makes stuff a lot simpler :)
To find a port address I fill a bunch of different-sized kalloc allocations with a pointer to the target port via mach messages using OOL_PORTS.

I then trigger the OOB read bug for various kalloc sizes and look for the most commonly leaked kernel pointer. Given the
semantics of kalloc this works well.

I make a pretty large number of kalloc allocations (via sending mach messages) in a kalloc size bin I won't use later, and I keep hold of them for now.

I allocate a bunch of mach ports to ensure that I have a page containing only my ports. I use the port address disclosure to find
a port which fits within particular bounds on a page. Once I've found it, I use the IOSurface bug to give myself a dangling pointer to that port.

I free the kalloc allocations made earlier and all the other ports then start making kalloc.4096 allocations (again via crafted mach messages.)

I do the reallocation slowly, 1MB at a time so that a kernel zone garbage collection will trigger and collect the page that the dangling pointer points to.

The GC will trigger when the zone map is over 95% full. It's easy to do that, the trick is to make sure there's plenty of stuff which the GC can collect
so that you don't get immediately killed by jetsam. All devices have the same sized zone map (384MB).

The replacement kalloc.4096 allocations are ipc_kmsg buffers which contain a fake IKOT_TASK port pointing to a fake struct task.
I use the bsdinfo->pid trick to build an arbitrary read with this (see details in async_wake.c.)

With the arbitrary read I find the kernel task's vm_map and the kernel ipc_space. I then free and reallocate the kalloc.4096 buffer replacing it with a fake
kernel task port.

Limitations:
The technique should work reliably enough for a security research tool. For me it works about 9/10 times. If you run it multiple times without rebooting,
it will probably panic, the GC forcing and reallocating trick isn't particularly advanced.

It's more likely to work after a fresh reboot.

The tfp0 returned by get_kernel_memory_rw should be safe to keep using after the exploit process has exited, but I haven't tested that.

## Porting to other devices:

Getting tfp0 should work for all devices running 11.1.2, it only requires structure offsets, not kernel symbols, which are unlikely to change between devices.
To port the PoC kernel debugger you need to find the correct symbols and update symbols.c, hints are given there.

For further discussion of this bug and other exploit techniques see:
http://blog.pangu.io/iosurfacerootuserclient-port-uaf/
https://siguza.github.io/v0rtex/
