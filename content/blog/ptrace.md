---
title: Rust systems programming - Implementing strace 
date: 2022-06-06T21:30:27+02:00
draft: true
---

While working on [lurk](https://github.com/JakWai01/lurk), I found it quite hard finding resources about working with `ptrace()` in Rust. 

There are some very helpful resources like this [blog](https://carstein.github.io/2022/05/29/rust-system-programming-2.html) by Michal Melewski or this [repository](https://github.com/upenn-cis198/homework4) by the University of Pennsylvania and the people behind these projects were super kind and open to answer questions. Still, I still feel like there is a lack of resources about the topic. 

In this article, we are building a small `strace` implementation in Rust using `ptrace()`. If you have any questions or feedback after reading the article, feel free to contact me on [Twitter](https://twitter.com/jakobwaibel).

# What is ptrace anyway?

The ptrace **man page** provides a solid definition: 

> The ptrace() system call provides a means by which one process (the "traceer") may observe or control the execution of another process (the "tracee"), and examine and change the tracee's memory and registers. It is primarily used to implement breakpoint debugging and system call tracing.

In other, simpler words: `ptrace()` allows us to interact with a process to set breakpoints to e.g. build a debugger (Yes, this is how gdb works) or to trace system calls (e.g. like in strace).

We can trace a process by setting up the calling process (e.g. our strace implementation) to be the parent process of the process we want to trace (e.g. an execution of ls) and then `ptrace()` allows us to see what our child process is doing. When interacting with the subprocess, the subprocess is stopped by using SIGTRAP until we allow it to continue.

For our purposes, we are going to use the `nix` crate to be able to use `ptrace()`. The `nix` crate generally provides various *nix system functions including `ptrace()`.

Like mentioned earlier, we will utilize `ptrace()` to build our own `strace` implementation. If you are rather interested in building a debugger, I suggest you read Michal Melewski's [blog](https://carstein.github.io/2022/05/29/rust-system-programming-2.html) instead.

# Tracing system calls by forking the calling process 

Generally, there are two approaches to trace system calls. Either we can attach to a running process or execute a command like `ls` in a process we created by forking our process. **Forking** a process essentially just means that we are creating a new process by duplicating the calling process. 

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

In the code snippet, we simply **fork** the calling process by calling the respective function. The `Parent` will be our calling process (tracer) and the `Child` will execute the command we want to trace (tracee). At this point, the parent and the child just run an infinite loop respectively.

When using top, we can detect that our process now consists out of two processes: 

```bash
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND          
  33778 jakob     20   0    3216   1008    896 R 100,0   0,0   0:12.98 ptrace           
  33803 jakob     20   0    3216    120      0 R 100,0   0,0   0:12.90 ptrace           
  24485 jakob     20   0  108,8g 485760 136464 S  21,6   1,5  11:57.68 chrome           
  25056 jakob     20   0   38,5g 254452 119724 S  19,3   0,8   2:30.57 codium           
   1930 jakob      9 -11 3409248  32552  23388 S   6,6   0,1   9:55.65 pulseaudio       
   2164 jakob     20   0 6941696 363176 176248 S   6,6   1,1  12:31.55 gnome-shell     
```

The first two processes in `top` are our two processes, both at 100% CPU usage. That's of course because both processes consist out of an inifinte loop. If you aren't using infinite loops inside the first code snippet, look for `ptrace` in the `COMMAND` column.

Additionally, we can also use `strace` to trace our implementation:

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

After using strace, we can see that the last system call executed is `clone`. 

Clone is defined like this: 

> "These system calls create a new ("child") process, in a manner similar to fork(2).". 

Exactly what was expected! Instead of just running an infinite loop, we can now use the child process to run a command we want to trace. 

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

To organize our code a bit better, I created a `run_tracer()` and `run_tracee()` function. The `run_tracer()` function is still running an inifite loop while the `run_tracee()` function is now executing `ls`.

```bash
$ cargo r
   Compiling ptrace v0.1.0 (/home/jakob/Documents/Projects/ptrace)
    Finished dev [unoptimized + debuginfo] target(s) in 0.36s
     Running `target/debug/ptrace`
Cargo.lock  Cargo.toml  src  target
```

When executing this snippet, we can see that `ls` is executed successfully. Be aware that the code will not exit automatically since the parent is running the infinite loop.

Now we should have all we need to trace some syscalls. Let's start by only tracing the first syscall that occurs.

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

Luckily, this code snippet is fairly short. 

First, we again `fork()` our process, use one of the processes as the `tracer()`, the other one as the `tracee()`. 

We will be using some of the functions Rust provides to interact with `ptrace()`. If you want to look at the original `ptrace()` specification to follow along, feel free to open up this [man page](https://linux.die.net/man/2/ptrace).

The `tracee()` simply has to confirm that it indeed wants to be traced. This can be achieved by calling `ptrace::traceme().unwrap()`. If you are looking at the description, search for `PTRACE_TRACEME`. In a real world application, please don't use `unwrap()` here. Afterwards, we use personality to disable **ASLR** (Address Space Layout Randomization). Then we execute `ls`.

The `tracer()` waits for a syscall using `wait()`. This function uses `waitpid()` to wait for the child process to change state. Look at the corresponding [man page](https://linux.die.net/man/2/waitpid) for further details. As soon as we get notified, we call `getregs()` (PTRACE_GETREGS) to get information about the general purpose or floating point registers of the `tracee`. This struct includes the system call number, the arguments of the syscall and it's return values. The **arguments** are stored in the following registers:

1. `rdi`
2. `rsi`
3. `rdx`
4. `r10`
5. `r8`
6. `r9`

The **system call number** is stored in `orig_rax` and the **return value** is stored in `rax` in the second invocation of the syscall. It is important to note that each system call triggers `wait()` twice. Once before being executed and once after. This can be useful to either modify arguments before the system call is executed or to get the return values after the syscall was executed. 

As we are just tracing the first syscall so far, we just want to know which one it is and don't care about its arguments. Mind that we only have the system call number by default. To display the corresponding name, we need some sort of a list containing the system calls for your architecture. The list I used can be found [here](https://gist.github.com/JakWai01/55049889b3a697010480b794a24befee) (x86_64). The system call number corresponds with the index in the list. 

```bash
$ cargo r
   Compiling ptrace v0.1.0 (/home/jakob/Documents/Projects/ptrace)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
     Running `target/debug/ptrace`
Syscall number: "execve"
```

When executing the snippet, we observe that the first syscall was `execve()`. This is not surprising at all. This system call is defined like this: "execve() executes the program referred to by pathname". This syscall is actually responsible for executing the program itself.

# Tracing multiple system calls

Tracing multiple system calls isn't actually that different from tracing a single one. 

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
    loop {
        wait().unwrap();

        match ptrace::getregs(child) {
            Ok(x) => println!(
                "Syscall number: {:?}",
                system_call_names::SYSTEM_CALL_NAMES[(x.orig_rax) as usize]
            ),
            Err(_) => break,
        };

        match ptrace::syscall(child, None) {
            Ok(_) => continue,
            Err(_) => break,
        }
    }
}

fn run_tracee() {
    ptrace::traceme().unwrap();
    personality(linux_personality::ADDR_NO_RANDOMIZE).unwrap();

    Command::new("ls").exec();

    exit(0)
}
```

As you can see in the code above, the only thing that changed is that we know run a loop in the tracer and call `ptrace::syscall()` at the end of the loop to continue execution until the next system call happens. This will result in all system calls being traced.

```bash
$ cargo r
   Compiling ptrace v0.1.0 (/home/jakob/Documents/Projects/ptrace)
    Finished dev [unoptimized + debuginfo] target(s) in 0.33s
     Running `target/debug/ptrace`
Syscall number: "execve"
Syscall number: "newfstatat"
Syscall number: "newfstatat"
Syscall number: "write"
Cargo.lock  Cargo.toml  src  target
Syscall number: "write"
Syscall number: "close"
Syscall number: "close"
Syscall number: "close"
Syscall number: "close"
Syscall number: "exit_group"
```

This output is shortened! Normally, it displays all the system calls involved in an execution of `ls`. 

That's it! We now have a very simple implementation of strace.

# Tracing system calls by attaching to a running process

After implementing the first approach to tracing system calls, the second one also isn't that different. 

```Rust

```

# Handling arguments of system calls

**Important**: There is no implementation provdided for this section. My solution most definitely is not worthy showing and would be too verbose for a single blog post. If you still want to look at the code, please go ahead, by clicking [this](https://github.com/JakWai01/lurk) link.

Reading the arguments of a system call in Rust is not that easy. We can read the arguments from the previously mentioned registers. The problem is that we don't know what type some argument has. I figured that in general, we need to distinguish three things: strings, integers and addresses. The problem is that all these types are represented as a simple integer in the register.

If the argument is a string we get a pointer to the beginning of the string. An address is just some number we want to display as hexadecimal later but we don't want to read the contents of. Integers are the easiest to handle. We can just use whatever is stored in that register. 

This problem had me struggling for several days. How can I figure out what number is an integer, an address or a string? The solution might be suprising. Suprinsingly stupid. I just annotated the type of every argument in the system call list. I still do not know if there is a hacky solution which would have saved me all my efforts (and probably mistakes). If you know a better solution, please let me know. I was just happy to have a working solution to my problem.

# Reading string arguments from memory

Reading strings from memory can be fairly complicated. Below is a function written by the colleagues of the University of Pennsylvania. You can find it in this [repository](https://github.com/upenn-cis198/homework4). This function can be used to read a string from the address provided in the corresponding register. Be aware that this function will return jibberish when not actually reading a string.

```Rust
fn read_string(pid: Pid, address: AddressType) -> String {
    let mut string = String::new();
    // Move 8 bytes up each time for next read.
    let mut count = 0;
    let word_size = 8;

    'done: loop {
        let mut bytes: Vec<u8> = vec![];
        let address = unsafe { address.offset(count) };

        let res: c_long;

        match ptrace::read(pid, address) {
            Ok(c_long) => res = c_long,
            Err(_) => break 'done,
        }

        bytes.write_i64::<LittleEndian>(res).unwrap_or_else(|err| {
            panic!("Failed to write {} as i64 LittleEndian: {}", res, err);
        });

        for b in bytes {
            if b != 0 {
                string.push(b as char);
            } else {
                break 'done;
            }
        }
        count += word_size;
    }

    string
}
```