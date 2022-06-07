---
title: "Who's your Daddy? - ptrace in Rust"
date: 2022-06-06T21:30:27+02:00
draft: true
---

A while ago, I was desperately searching for resources on how to use ptrace in Rust. There are some helpful resources like @carsteins [blog](https://carstein.github.io/2022/05/29/rust-system-programming-2.html)  or this [repository](https://github.com/upenn-cis198/homework4). However, after looking at these resources, I still had questions about the topic. My goal with this post is to answer some of the questions that usually occur when working with ptrace by implementing a simple version of strace.

# What is ptrace anyway?

The ptrace man page provides a solid definition: "The ptrace() system call provides a means by which one process (the "traceer") may observe or control the execution of another process (the "tracee"), and examine and change the tracee's memory and registers. It is primarily used to implement breakpoint debugging and system call tracing.

In other words: ptrace() allows us to interact with a process to allow us to set breakpoints to e.g. build a debugger (Yes, this is how gdb works) or to trace system calls (e.g. strace). 

We can trace a process by making the calling process (e.g. our strace implementation) the parent process of the process we want to trace and then ptrace() allows us to see what our child process is doing.

![test](https://media.makeameme.org/created/look-at-me-yxw5ov.jpg)

In this article, we will utilize ptrace() to build our own strace implementation. If you are rather interested in building a debugger, I suggest you read @carsteins [blog](https://carstein.github.io/2022/05/29/rust-system-programming-2.html) instead.

# Tracing system calls using ptrace

Generally, there are two approaches to trace system calls. Either we can attach to a running process or execute a command like `ls` in a process we created by forking our process. Let's have a look at a simple example: 

```
Hello, World!
```

In the code snippet, we simply **fork** the calling process. **Forking** a process essentially just means that we are creating a new process by duplicating the calling process. One of the two processes will later be used as the "tracer" and the other one will become the "tracee".

```
strace
```

When we use strace on our code snippet above, we can see that the last system call executed is `clone` which indicates that this process was split into two subprocesses. Let's have a look what our processes are!

```
Look what Pids we have
```

We could now use the child process to run a command we want to trace. Mind that `Command.exec()` will continue in this process, while `Command.spawn()` would continue executing in some other process, which is not what we aim for.

```
Implement Command.exec() and see that nothing but fork and execute ls in the child process.
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

# Handling arguments of system calls

**Important**: There is no implementation provdided for this section. My solution most definitely is not worthy showing and would be too verbose for a single blog post. If you still want to look at the code, please go ahead, by clicking [this](https://github.com/JakWai01/lurk) link.

Reading the arguments of a system call is not that easy. We have read the arguments from the previously mentioned registers in memory. The problem is that we don't know what type an argument has. I figured that in general, we need to distinguish three things: strings, integers and addresses. And the problem is: THEY ALL ARE INTEGERS.

If the argument is a string we get a pointer to the beginning of the string. This pointer is an INTEGER. An address looks like a string (so it's and INTEGER) in memory but we don't want to read the contents of that address. Integers are the easiest to handle. We can just use whatever is stored in that register (AN INTEGER). 

This problem had me struggling for several days. How can I figure out what number is an Integer, an address or a string, since I had to handle them differently. The solution might be suprising. Suprinsingly stupid! I just annotated the type of every argument in the system call list. I still do not know if there is a hacky solution which would have saved me all my efforts (and probably mistakes). If you know a better solution, please let me know. I was just happy to have a working solution to my problem.

That's it!
