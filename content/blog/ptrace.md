---
title: "Who's your Daddy? - ptrace in Rust"
date: 2022-06-06T21:30:27+02:00
draft: true
---

A while ago, I was desperately searching for resources on how to use ptrace in Rust. There are some helpful resources like @carsteins [blog](https://carstein.github.io/2022/05/29/rust-system-programming-2.html)  or this [repository](https://github.com/upenn-cis198/homework4). However, after looking at these resources, I still had questions about the topic. My goal with this post is to answer some of the questions that usually occur when working with ptrace by implementing a simple version of strace.

# What is ptrace anyway?

The ptrace man page provides a solid definition: "The ptrace() system call provides a means by which one process (the "traceer") may observe or control the execution of another process (the "tracee"), and examine and change the tracee's memory and registers. It is primarily used to implement breakpoint debugging and system call tracing.

In other words: ptrace() allows us to interact with a process to set breakpoints to e.g. build a debugger (Yes, this is how gdb works) or to trace system calls (e.g. strace).

We can trace a process by making the calling process (e.g. our strace implementation) the parent process of the process we want to trace and then ptrace() allows us to see what our child process is doing.

![test](https://media.makeameme.org/created/look-at-me-yxw5ov.jpg)

In this article, we will utilize ptrace() to build our own strace implementation. If you are rather interested in building a debugger, I suggest you read @carsteins [blog](https://carstein.github.io/2022/05/29/rust-system-programming-2.html) instead.

# Tracing system calls using ptrace

Generally, there are two approaches to trace system calls. Either we can attach to a running process or execute a command like `ls` in a process we created by forking our process. 

First let's have a look on how to fork a process to create a child process:

```Rust
use nix::unistd::{fork, ForkResult};

fn main() {
    match unsafe { fork() } {
        Ok(ForkResult::Child) => {
            loop{}
        } 

        Ok(ForkResult::Parent { child: _ }) => {
            loop{}
        }

        Err(err) => {
            panic!("[main] fork() failed: {}", err);
        }
    }
}
```

In the code snippet, we simply **fork** the calling process by calling the respective function. **Forking** a process essentially just means that we are creating a new process by duplicating the calling process. The `Parent` will be our calling process (tracer) and the `Child` will execute the command we want to trace (tracee).

We can see that our process now consists out of two processes by using `top`:

```bash
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND          
  33778 jakob     20   0    3216   1008    896 R 100,0   0,0   0:12.98 ptrace           
  33803 jakob     20   0    3216    120      0 R 100,0   0,0   0:12.90 ptrace           
  24485 jakob     20   0  108,8g 485760 136464 S  21,6   1,5  11:57.68 chrome           
  25056 jakob     20   0   38,5g 254452 119724 S  19,3   0,8   2:30.57 codium           
   1930 jakob      9 -11 3409248  32552  23388 S   6,6   0,1   9:55.65 pulseaudio       
   2164 jakob     20   0 6941696 363176 176248 S   6,6   1,1  12:31.55 gnome-shell     
```

The first two processes in `top` are our two processes, both at 100% CPU usage. That's of course because both processes just consist out of an inifinte loop respectively.

We can also use `strace` to inspect what we have done:

```bash
$ strace cargo r
[...]
brk(NULL)                               = 0x559a6ac5f000
brk(0x559a6ac80000)                     = 0x559a6ac80000
openat(AT_FDCWD, "/proc/self/maps", O_RDONLY|O_CLOEXEC) = 3
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
newfstatat(3, "", {st_mode=S_IFREG|0444, st_size=0, ...}, AT_EMPTY_PATH) = 0
read(3, "559a6a59e000-559a6a5a4000 r--p 0"..., 1024) = 1024
read(3, "000 r--p 001bd000 103:02 2365710"..., 1024) = 1024
read(3, "3a000-7fb34513c000 r--p 00000000"..., 1024) = 918
close(3)                                = 0
sched_getaffinity(34776, 32, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15]) = 8
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7fb344edea50) = 34790
```

When we use strace on our code snippet above, we can see that the last system call executed is `clone`. Clone is defined as: "These system calls create a new ("child") process, in a manner
similar to fork(2).". Exactly what we expected!

We can now use the child process to run a command we want to trace. Mind that `Command.exec()` will continue in this process, while `Command.spawn()` would continue executing in some other process, which is not what we aim for.

```Rust
use nix::unistd::{fork, ForkResult, Pid};
use std::os::unix::process::CommandExt;
use std::process::Command;

fn main() {
    match unsafe { fork() } {
        Ok(ForkResult::Child) => {
            run_tracee();
        } 

        Ok(ForkResult::Parent { child }) => {
            run_tracer(child);
        }

        Err(err) => {
            panic!("[main] fork() failed: {}", err);
        }
    }
}

fn run_tracer(_child: Pid) {
    loop{}
}

fn run_tracee() {
    Command::new("ls").exec();
}
```

When executing this snippet, we can see that ls is executed successfully. Be aware that the code will not exit automatically since the parent is running the infinite loop:

```bash
$ cargo r
   Compiling ptrace v0.1.0 (/home/jakob/Documents/Projects/ptrace)
    Finished dev [unoptimized + debuginfo] target(s) in 0.36s
     Running `target/debug/ptrace`
Cargo.lock  Cargo.toml  src  target
```

Now we should have all we need to trace some syscalls. Let's start by only tracing the first syscall that occurs! 

```Rust
mod system_call_names;

use linux_personality::personality;
use nix::sys::ptrace;
use nix::sys::wait::wait;
use nix::unistd::{fork, ForkResult, Pid};
use std::os::unix::process::CommandExt;
use std::process::{exit, Command};

fn main() {
    match unsafe { fork() } {
        Ok(ForkResult::Child) => {
            run_tracee();
        }

        Ok(ForkResult::Parent { child }) => {
            run_tracer(child);
        }

        Err(err) => {
            panic!("[main] fork() failed: {}", err);
        }
    }
}

fn run_tracer(child: Pid) {
    wait().unwrap();

    match ptrace::getregs(child) {
        Ok(x) => println!(
            "Syscall number: {:?}",
            system_call_names::SYSTEM_CALL_NAMES[(x.orig_rax) as usize]
        ),
        Err(x) => println!("{:?}", x),
    };
}

fn run_tracee() {
    ptrace::traceme().unwrap();
    personality(linux_personality::ADDR_NO_RANDOMIZE).unwrap();

    Command::new("ls").exec();

    exit(0)
}
```

Luckily, this code snippet is fairly simple. 

First, we need to `fork()` our process, use one of the processes as the `tracer()`, the other one as the `tracee()`. 

The `tracee()` simply has to confirm that it indeed wants to be traced. This can be achieved by calling `ptrace::traceme().unwrap()`. In a real world application, please don't use `unwrap()` here. We use personality to disable **ASLR** (Address Space Layout Randomization). We do this, so that the address space is not randomly arranged. Then we execute ls.

The `tracer()` waits for a syscall using the `wait()` function. When a system call is being called, we call the `getregs()` function to get information about the system call as a struct (user_regs_struct). This struct includes the system call number, the arguments of the syscall and it's return values. The **arguments** are stored in the following registers:

1. `rdi`
2. `rsi`
3. `rdx`
4. `r10`
5. `r8`
6. `r9`

The **system call number** is stored in `orig_rax` and the **return value** is stored in `rax` in the second invocation of the syscall (see next paragraph). 

**Important**: Each system call triggers wait() twice. Once before being executed and once after. This can be useful to either modify arguments before the system call is executed or see return values after the syscall was executed. 

We then simply print the current system call. Mind that we only have the system call number by default. If we want to display the name. We need some sort of a list containing the system calls for your architecture. The list I used can be found [here](https://gist.github.com/JakWai01/55049889b3a697010480b794a24befee) (x86_64).

Executing this snippet yields the following:

```bash
$ cargo r
   Compiling ptrace v0.1.0 (/home/jakob/Documents/Projects/ptrace)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
     Running `target/debug/ptrace`
Syscall number: "execve"
```

As we can see, the first syscall was `execve`. This is not surprising at all. It is defined like this: "execve() executes the program referred to by pathname". This syscall is actually responsible for executing the program itself.

# Handling arguments of system calls

**Important**: There is no implementation provdided for this section. My solution most definitely is not worthy showing and would be too verbose for a single blog post. If you still want to look at the code, please go ahead, by clicking [this](https://github.com/JakWai01/lurk) link.

Reading the arguments of a system call is not that easy. We have read the arguments from the previously mentioned registers in memory. The problem is that we don't know what type an argument has. I figured that in general, we need to distinguish three things: strings, integers and addresses. And the problem is: THEY ALL ARE INTEGERS.

If the argument is a string we get a pointer to the beginning of the string. This pointer is an INTEGER. An address looks like a string (so it's and INTEGER) in memory but we don't want to read the contents of that address. Integers are the easiest to handle. We can just use whatever is stored in that register (AN INTEGER). 

This problem had me struggling for several days. How can I figure out what number is an Integer, an address or a string, since I had to handle them differently. The solution might be suprising. Suprinsingly stupid! I just annotated the type of every argument in the system call list. I still do not know if there is a hacky solution which would have saved me all my efforts (and probably mistakes). If you know a better solution, please let me know. I was just happy to have a working solution to my problem.

That's it!
