# Program type `BPF_PROG_TYPE_TRACING`

<!-- [FEATURE_TAG](BPF_PROG_TYPE_TRACING) -->
[:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/f1b9509c2fb0ef4db8d22dac9aef8e856a5d81f6)
<!-- [/FEATURE_TAG] -->

Tracing programs are a newer alternative to kprobes and tracepoints. Tracing programs utilize BPF trampolines, a new mechanism which provides practically zero overhead. In addition, tracing programs can be attached to BPF programs to provide troubleshooting and debugging capabilities, something that is not possible with kprobes.

## Usage

There are a few variations of tracing programs depending on their attach type. 

### Raw tracepoint

<!-- [FEATURE_TAG](BPF_TRACE_RAW_TP) -->
[:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/f1b9509c2fb0ef4db8d22dac9aef8e856a5d81f6)
<!-- [/FEATURE_TAG] -->

Raw tracepoint programs can be loaded as its own dedicated program type or as an attach type under the tracing program type. When loaded as a tracing program it can attach to a BTF ID of a tracepoint via a link instead of having to use the special syscall to attach.

For details see the [Raw Tracepoint](BPF_PROG_TYPE_RAW_TRACEPOINT.md) page.

### Fentry

<!-- [FEATURE_TAG](BPF_TRACE_FENTRY) -->
[:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/fec56f5890d93fc2ed74166c397dc186b1c25951)
<!-- [/FEATURE_TAG] -->

Fentry programs are similar in function to a kprobe attached to a functions first instruction. This program type is invoked before control passes to the function to allow for tracing/observation.

Kprobes do not have to be attached at the entry point of a function, kprobes can be installed at any point in the function, whereas fentry programs are always attached at the entry point of a function.

Fentry programs are attached to a [BPF trampoline](../concepts/bpf-trampoline.md) which causes less overhead than kprobes. Fentry programs can also be attached to BPF programs such as XDP, TC or cgroup programs which makes debugging eBPF programs easier. Kprobes lack this capability.

Fentry programs are typically located in an ELF section prefixed with `fentry/`.

### Fexit

<!-- [FEATURE_TAG](BPF_TRACE_FEXIT) -->
[:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/fec56f5890d93fc2ed74166c397dc186b1c25951)
<!-- [/FEATURE_TAG] -->

Fexit programs are similar to kretprobes. The program is invoked when the function returns no matter where the return occurs. Fexit programs get invoked with the input arguments and the return value of the function, so there is no need to store the input arguments in a map like you would have to do with kprobes and kretprobes.

Fexit programs are typically located in an ELF section prefixed with `fexit/`.

### Modify return

<!-- [FEATURE_TAG](BPF_MODIFY_RETURN) -->
[:octicons-tag-24: v5.7](https://github.com/torvalds/linux/commit/ae24082331d9bbaae283aafbe930a8f0eb85605a)
<!-- [/FEATURE_TAG] -->

Fmodify_return programs run after the fentry program but before the function we are tracing. Unlike the fentry and fexit programs, the fmodify_return program can return non-zero values. When a non-zero value is returned, the function we are tracing will not be executed and the value returned by the fmodify_return program will be returned instead.

Fmodify_return programs are provided with the input arguments to the function under trace and a return value. If multiple fmodify_return programs are attached, then the return value of the previous fmodify_return program will be provided as the input to the next fmodify_return program.

Unlike fentry/fexit programs, fmodify_return programs are only allowed for security hooks (with an extra `CAP_MAC_ADMIN` check) and functions whitelisted for error injection (`ALLOW_ERROR_INJECTION`).   
<!-- TODO how does this work with HID-BPF -->

Fmodify_return programs are typically located in an ELF section prefixed with `fmod_ret/`.

### Iterator

<!-- [FEATURE_TAG](BPF_TRACE_ITER) -->
[:octicons-tag-24: v5.8](https://github.com/torvalds/linux/commit/15d83c4d7cef5c067a8b075ce59e97df4f60706e)
<!-- [/FEATURE_TAG] -->

Iterator programs use the same program type but have a different use case. Iterator programs are used to iterate over a list of in kernel data structures to efficiently collect data and/or to summarize data, specifically in cases where otherwise the kernel-userspace boundary would cause a bottleneck.

Iterator programs can only be attached to specific pre-defined iterators. Each iterator follows the naming convention `bpf_iter__<iter_name>` which is a type which can be found in the vmlinux of the kernel. This type is also the context with which the program will be invoked for each data structure in the iterator.

Iterator programs are typically located in an ELF section prefixed with `iter/`.

## Context

### Raw tracepoint

Please see the [Raw Tracepoint](BPF_PROG_TYPE_RAW_TRACEPOINT.md) page for details.

### Fentry / Fexit / Fmodify_return

Programs for all of these attach types are provided with an array of u64 values representing the arguments to the function that is being traced. The Fexit and Fmodify_return programs are also provided with the return value of the function or the previous Fmodify_return program.

The `BPF_PROG` and `BPF_PROG2` macros defined in libbpf can be used to cast the values of the array to their proper types to provide a more natural way of declaring a BPF program.

Some functions in the kernel are passed structures which are larger than 8 bytes, in that case the value of the argument may be spread over multiple indexes in the array. The `BPF_PROG` cannot deal with this, so when writing BPF programs that may attach to functions which take structures as arguments, the `BPF_PROG2` macro should be used instead.

### Iterator

The context for iterator programs differs per iterator, however, the first field of every iterator context `meta` is a pointer to `struct bpf_iter_meta`:

```c
struct bpf_iter_meta {
	struct seq_file *seq;
	__u64 session_id;
	__u64 seq_num;
};
```

The rest of the context struct will contain the data structure for the current iteration.

The metadata contains a sequence file, which is effectively the output of the iterator program. When a pinned iterator is read, the iterator program will be invoked for each data structure in the iterator. The iterator program can then use the seq_file to output data to the user. Dedicated print helpers are used to write to the sequence file such as [`bpf_seq_printf`](../helper-function/bpf_seq_printf.md), [`bpf_seq_printf_btf`](../helper-function/bpf_seq_printf_btf.md), and [`bpf_seq_write`](../helper-function/bpf_seq_write.md).

The `session_id` is an incrementing value which is used to identify the current iteration session. The `seq_num` is the current iteration number within the session.

!!! example "Docs could be improved"
    This part of the docs is incomplete, contributions are very welcome
    <!-- Iterators have a lot of edge cases and are frankly to big to be lumped in with fentry/fexit programs, the deserve their own page even though its the same attach type -->

## Attachment

All tracing programs are attached via [BPF links](../syscall/BPF_LINK_CREATE.md). The program should be loaded with the correct attach type and the same attach type used when creating the link via the `attach_type` attribute. The tracepoint, function or iterator to attach to should be specified via the `target_btf_id` attribute, its value matching the BTF ID for the target from the vmlinux BTF blob.

## Example

??? example "Raw tracepoint"
    ```c
    /**
    * A trivial example tracepoint program that shows how to
    * acquire and release a struct task_struct * pointer.
    */
    SEC("tp_btf/task_newtask")
    int BPF_PROG(task_acquire_release_example, struct task_struct *task, u64 clone_flags)
    {
        struct task_struct *acquired;

        acquired = bpf_task_acquire(task);
        if (acquired)
            /*
                * In a typical program you'd do something like store
                * the task in a map, and the map will automatically
                * release it later. Here, we release it manually.
                */
            bpf_task_release(acquired);
        return 0;
    }
    ```

??? example "Fentry"
    ```c
    // SPDX-License-Identifier: GPL-2.0

    #include "vmlinux.h"
    #include <bpf/bpf_helpers.h>
    #include <bpf/bpf_tracing.h>

    extern const int bpf_prog_active __ksym;

    struct {
        __uint(type, BPF_MAP_TYPE_RINGBUF);
        __uint(max_entries, 1 << 12);
    } ringbuf SEC(".maps");

    SEC("fentry/security_inode_getattr")
    int BPF_PROG(d_path_check_rdonly_mem, struct path *path, struct kstat *stat,
            __u32 request_mask, unsigned int query_flags)
    {
        void *active;
        u32 cpu;

        cpu = bpf_get_smp_processor_id();
        active = (void *)bpf_per_cpu_ptr(&bpf_prog_active, cpu);
        if (active) {
            /* FAIL here! 'active' points to 'regular' memory. It
            * cannot be submitted to ring buffer.
            */
            bpf_ringbuf_submit(active, 0);
        }
        return 0;
    }

    char _license[] SEC("license") = "GPL";
    ```

??? example "Fexit"
    ```c
    SEC("fexit/inet_stream_connect")
    int BPF_PROG(update_cookie_tracing, struct socket *sock,
            struct sockaddr *uaddr, int addr_len, int flags, int ret)
    {
        struct socket_cookie *p;

        if (uaddr->sa_family != AF_INET6)
            return 0;

        p = bpf_cgrp_storage_get(&socket_cookies, sock->sk->sk_cgrp_data.cgroup, 0, 0);
        if (!p)
            return 0;

        if (p->cookie_key != bpf_get_socket_cookie(sock->sk))
            return 0;

        p->cookie_value |= 0xF0;
        return 0;
    }
    ```

??? example "Fmodify_return"
    ```c
    // SPDX-License-Identifier: GPL-2.0

    #include "vmlinux.h"
    #include <bpf/bpf_helpers.h>
    #include <bpf/bpf_tracing.h>
    #include "hid_bpf_helpers.h"

    SEC("fmod_ret/hid_bpf_device_event")
    int BPF_PROG(hid_y_event, struct hid_bpf_ctx *hctx)
    {
        s16 y;
        __u8 *data = hid_bpf_get_data(hctx, 0 /* offset */, 9 /* size */);

        if (!data)
            return 0; /* EPERM check */

        bpf_printk("event: size: %d", hctx->size);
        bpf_printk("incoming event: %02x %02x %02x",
            data[0],
            data[1],
            data[2]);
        bpf_printk("                %02x %02x %02x",
            data[3],
            data[4],
            data[5]);
        bpf_printk("                %02x %02x %02x",
            data[6],
            data[7],
            data[8]);

        y = data[3] | (data[4] << 8);

        y = -y;

        data[3] = y & 0xFF;
        data[4] = (y >> 8) & 0xFF;

        bpf_printk("modified event: %02x %02x %02x",
            data[0],
            data[1],
            data[2]);
        bpf_printk("                %02x %02x %02x",
            data[3],
            data[4],
            data[5]);
        bpf_printk("                %02x %02x %02x",
            data[6],
            data[7],
            data[8]);

        return 0;
    }

    SEC("fmod_ret/hid_bpf_device_event")
    int BPF_PROG(hid_x_event, struct hid_bpf_ctx *hctx)
    {
        s16 x;
        __u8 *data = hid_bpf_get_data(hctx, 0 /* offset */, 9 /* size */);

        if (!data)
            return 0; /* EPERM check */

        x = data[1] | (data[2] << 8);

        x = -x;

        data[1] = x & 0xFF;
        data[2] = (x >> 8) & 0xFF;
        return 0;
    }

    SEC("fmod_ret/hid_bpf_rdesc_fixup")
    int BPF_PROG(hid_rdesc_fixup, struct hid_bpf_ctx *hctx)
    {
        __u8 *data = hid_bpf_get_data(hctx, 0 /* offset */, 4096 /* size */);

        if (!data)
            return 0; /* EPERM check */

        bpf_printk("rdesc: %02x %02x %02x",
            data[0],
            data[1],
            data[2]);
        bpf_printk("       %02x %02x %02x",
            data[3],
            data[4],
            data[5]);
        bpf_printk("       %02x %02x %02x ...",
            data[6],
            data[7],
            data[8]);

        /*
        * The original report descriptor contains:
        *
        * 0x05, 0x01,                    //   Usage Page (Generic Desktop)      30
        * 0x16, 0x01, 0x80,              //   Logical Minimum (-32767)          32
        * 0x26, 0xff, 0x7f,              //   Logical Maximum (32767)           35
        * 0x09, 0x30,                    //   Usage (X)                         38
        * 0x09, 0x31,                    //   Usage (Y)                         40
        *
        * So byte 39 contains Usage X and byte 41 Usage Y.
        *
        * We simply swap the axes here.
        */
        data[39] = 0x31;
        data[41] = 0x30;

        return 0;
    }

    char _license[] SEC("license") = "GPL";
    ```

??? example "Iterator"
    ```c
    SEC("iter/task_file")
    int dump_task_file(struct bpf_iter__task_file *ctx)
    {
        struct seq_file *seq = ctx->meta->seq;
        struct task_struct *task = ctx->task;
        struct file *file = ctx->file;
        __u32 fd = ctx->fd;

        if (task == NULL || file == NULL)
            return 0;

        if (ctx->meta->seq_num == 0) {
            count = 0;
            BPF_SEQ_PRINTF(seq, "    tgid      gid       fd      file\n");
        }

        if (tgid == task->tgid && task->tgid != task->pid)
            count++;

        if (last_tgid != task->tgid) {
            last_tgid = task->tgid;
            unique_tgid_count++;
        }

        BPF_SEQ_PRINTF(seq, "%8d %8d %8d %lx\n", task->tgid, task->pid, fd,
                (long)file->f_op);
        return 0;
    }
    ```

## Helper functions

Not all helper functions are available in all program types. These are the helper calls available for raw tracepoint programs:

<!-- DO NOT EDIT MANUALLY -->
<!-- [PROG_HELPER_FUNC_REF] -->
??? abstract "Supported helper functions"
    * [bpf_map_lookup_elem](../helper-function/bpf_map_lookup_elem.md)
    * [bpf_map_update_elem](../helper-function/bpf_map_update_elem.md)
    * [bpf_map_delete_elem](../helper-function/bpf_map_delete_elem.md)
    * [bpf_map_push_elem](../helper-function/bpf_map_push_elem.md)
    * [bpf_map_pop_elem](../helper-function/bpf_map_pop_elem.md)
    * [bpf_map_peek_elem](../helper-function/bpf_map_peek_elem.md)
    * [bpf_map_lookup_percpu_elem](../helper-function/bpf_map_lookup_percpu_elem.md)
    * [bpf_ktime_get_ns](../helper-function/bpf_ktime_get_ns.md)
    * [bpf_ktime_get_boot_ns](../helper-function/bpf_ktime_get_boot_ns.md)
    * [bpf_tail_call](../helper-function/bpf_tail_call.md)
    * [bpf_get_current_pid_tgid](../helper-function/bpf_get_current_pid_tgid.md)
    * [bpf_get_current_task](../helper-function/bpf_get_current_task.md)
    * [bpf_get_current_task_btf](../helper-function/bpf_get_current_task_btf.md)
    * [bpf_task_pt_regs](../helper-function/bpf_task_pt_regs.md)
    * [bpf_get_current_uid_gid](../helper-function/bpf_get_current_uid_gid.md)
    * [bpf_get_current_comm](../helper-function/bpf_get_current_comm.md)
    * [bpf_trace_printk](../helper-function/bpf_trace_printk.md)
    * [bpf_get_smp_processor_id](../helper-function/bpf_get_smp_processor_id.md)
    * [bpf_get_numa_node_id](../helper-function/bpf_get_numa_node_id.md)
    * [bpf_perf_event_read](../helper-function/bpf_perf_event_read.md)
    * [bpf_current_task_under_cgroup](../helper-function/bpf_current_task_under_cgroup.md)
    * [bpf_get_prandom_u32](../helper-function/bpf_get_prandom_u32.md)
    * [bpf_probe_write_user](../helper-function/bpf_probe_write_user.md)
    * [bpf_probe_read_user](../helper-function/bpf_probe_read_user.md)
    * [bpf_probe_read_kernel](../helper-function/bpf_probe_read_kernel.md)
    * [bpf_probe_read_user_str](../helper-function/bpf_probe_read_user_str.md)
    * [bpf_probe_read_kernel_str](../helper-function/bpf_probe_read_kernel_str.md)
    * [bpf_probe_read](../helper-function/bpf_probe_read.md)
    * [bpf_probe_read_str](../helper-function/bpf_probe_read_str.md)
    * [bpf_get_current_cgroup_id](../helper-function/bpf_get_current_cgroup_id.md)
    * [bpf_get_current_ancestor_cgroup_id](../helper-function/bpf_get_current_ancestor_cgroup_id.md)
    * [bpf_send_signal](../helper-function/bpf_send_signal.md)
    * [bpf_send_signal_thread](../helper-function/bpf_send_signal_thread.md)
    * [bpf_perf_event_read_value](../helper-function/bpf_perf_event_read_value.md)
    * [bpf_get_ns_current_pid_tgid](../helper-function/bpf_get_ns_current_pid_tgid.md)
    * [bpf_ringbuf_output](../helper-function/bpf_ringbuf_output.md)
    * [bpf_ringbuf_reserve](../helper-function/bpf_ringbuf_reserve.md)
    * [bpf_ringbuf_submit](../helper-function/bpf_ringbuf_submit.md)
    * [bpf_ringbuf_discard](../helper-function/bpf_ringbuf_discard.md)
    * [bpf_ringbuf_query](../helper-function/bpf_ringbuf_query.md)
    * [bpf_jiffies64](../helper-function/bpf_jiffies64.md)
    * [bpf_get_task_stack](../helper-function/bpf_get_task_stack.md)
    * [bpf_copy_from_user](../helper-function/bpf_copy_from_user.md)
    * [bpf_copy_from_user_task](../helper-function/bpf_copy_from_user_task.md)
    * [bpf_snprintf_btf](../helper-function/bpf_snprintf_btf.md)
    * [bpf_per_cpu_ptr](../helper-function/bpf_per_cpu_ptr.md)
    * [bpf_this_cpu_ptr](../helper-function/bpf_this_cpu_ptr.md)
    * [bpf_task_storage_get](../helper-function/bpf_task_storage_get.md)
    * [bpf_task_storage_delete](../helper-function/bpf_task_storage_delete.md)
    * [bpf_for_each_map_elem](../helper-function/bpf_for_each_map_elem.md)
    * [bpf_snprintf](../helper-function/bpf_snprintf.md)
    * [bpf_get_func_ip](../helper-function/bpf_get_func_ip.md)
    * [bpf_get_branch_snapshot](../helper-function/bpf_get_branch_snapshot.md)
    * [bpf_find_vma](../helper-function/bpf_find_vma.md)
    * [bpf_trace_vprintk](../helper-function/bpf_trace_vprintk.md)
    * [bpf_map_lookup_elem](../helper-function/bpf_map_lookup_elem.md)
    * [bpf_map_update_elem](../helper-function/bpf_map_update_elem.md)
    * [bpf_map_delete_elem](../helper-function/bpf_map_delete_elem.md)
    * [bpf_map_push_elem](../helper-function/bpf_map_push_elem.md)
    * [bpf_map_pop_elem](../helper-function/bpf_map_pop_elem.md)
    * [bpf_map_peek_elem](../helper-function/bpf_map_peek_elem.md)
    * [bpf_map_lookup_percpu_elem](../helper-function/bpf_map_lookup_percpu_elem.md)
    * [bpf_get_prandom_u32](../helper-function/bpf_get_prandom_u32.md)
    * [bpf_get_smp_processor_id](../helper-function/bpf_get_smp_processor_id.md)
    * [bpf_get_numa_node_id](../helper-function/bpf_get_numa_node_id.md)
    * [bpf_tail_call](../helper-function/bpf_tail_call.md)
    * [bpf_ktime_get_ns](../helper-function/bpf_ktime_get_ns.md)
    * [bpf_ktime_get_boot_ns](../helper-function/bpf_ktime_get_boot_ns.md)
    * [bpf_ringbuf_output](../helper-function/bpf_ringbuf_output.md)
    * [bpf_ringbuf_reserve](../helper-function/bpf_ringbuf_reserve.md)
    * [bpf_ringbuf_submit](../helper-function/bpf_ringbuf_submit.md)
    * [bpf_ringbuf_discard](../helper-function/bpf_ringbuf_discard.md)
    * [bpf_ringbuf_query](../helper-function/bpf_ringbuf_query.md)
    * [bpf_for_each_map_elem](../helper-function/bpf_for_each_map_elem.md)
    * [bpf_loop](../helper-function/bpf_loop.md)
    * [bpf_strncmp](../helper-function/bpf_strncmp.md)
    * [bpf_spin_lock](../helper-function/bpf_spin_lock.md)
    * [bpf_spin_unlock](../helper-function/bpf_spin_unlock.md)
    * [bpf_jiffies64](../helper-function/bpf_jiffies64.md)
    * [bpf_per_cpu_ptr](../helper-function/bpf_per_cpu_ptr.md)
    * [bpf_this_cpu_ptr](../helper-function/bpf_this_cpu_ptr.md)
    * [bpf_timer_init](../helper-function/bpf_timer_init.md)
    * [bpf_timer_set_callback](../helper-function/bpf_timer_set_callback.md)
    * [bpf_timer_start](../helper-function/bpf_timer_start.md)
    * [bpf_timer_cancel](../helper-function/bpf_timer_cancel.md)
    * [bpf_trace_printk](../helper-function/bpf_trace_printk.md)
    * [bpf_get_current_task](../helper-function/bpf_get_current_task.md)
    * [bpf_get_current_task_btf](../helper-function/bpf_get_current_task_btf.md)
    * [bpf_probe_read_user](../helper-function/bpf_probe_read_user.md)
    * [bpf_probe_read_kernel](../helper-function/bpf_probe_read_kernel.md)
    * [bpf_probe_read_user_str](../helper-function/bpf_probe_read_user_str.md)
    * [bpf_probe_read_kernel_str](../helper-function/bpf_probe_read_kernel_str.md)
    * [bpf_snprintf_btf](../helper-function/bpf_snprintf_btf.md)
    * [bpf_snprintf](../helper-function/bpf_snprintf.md)
    * [bpf_task_pt_regs](../helper-function/bpf_task_pt_regs.md)
    * [bpf_trace_vprintk](../helper-function/bpf_trace_vprintk.md)
<!-- [/PROG_HELPER_FUNC_REF] -->
