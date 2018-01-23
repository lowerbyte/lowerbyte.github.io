---
layout: post
title: Lbdb debugger - introduction to my PoC debugger.
---

So how those all debuggers works?! The best way to find out is to write one!

_Note: I'm also learning. If you find any wrong or misleading information, please write to me._  
_Note: All the code you may find at my github in lbdb\_debugger repository_

I know, I know, I know...  
I promised to cover Python Internals Part 2 and you guys have to forgive me, but when I got this idea it couldn't wait.  
During the Christmas time and debugging another not working program I got this thought: _How does it work? How can I stop the execution, set the breakpoint and how the hell it knows where to put it, when I write 'break main'_. Then I decided to write one, but due to all responibilities I found the time lately.  
So what's this series will cover? I must admit I do not have a strict plan but for sure we will discuss `ptrace`, I will try to guide you trough the `DWARF` (don't worry I will explain it later), how `breakpoints` are set and how to disable them.

In this introduction I will start from the very beginning - the very powerful system call **ptrace**!  
What tells us the Wikipedia?  

>In computing, a system call is the programmatic way in which a computer program requests a service from the kernel of the operating system it is executed on.

And that's it and that's it! _Request a service from the kernel_ sounds very nice, so now it's time to make friends with the `ptrace`!  
**ptrace** - _process trace_ according to the man page is:

>The  ptrace()  system  call  provides a means by which one process (the "tracer") may observe and control the execution of another process (the "tracee"),  and  examine  and change the tracee's memory and registers. It is primarily used to implement breakpoint debugging and system  call tracing.

So it seems like the right tool for this job. Basically all information which are useful are in this man page, so just write `man 2 ptrace` and read it. I will paste only short cuttings from it.

>A process can initiate a  trace  by  calling  fork(2)  and  having  the resulting  child  do  a  PTRACE_TRACEME,  followed  (typically)  by  an execve(2).

Ok so to trace a process we may `fork` it. Great, sure, but what the `fork` is? And again my friend - the **man page**! Let's hit `man fork`:

>fork()  creates  a new process by duplicating the calling process.  The new process is referred to as the child process.  The  calling  process is referred to as the parent process. The child process and the parent process run in separate memory spaces. At the time of fork() both memory spaces have the same content.  
(...)  
On success, the PID of the child process is returned in the parent, and 0 is returned in the child.  On failure, -1 is returned in the  parent, no child process is created, and errno is set appropriately.

Ooo it makes sense now! It time to start coding.

```c
//some skipped code
int main(int argv, char** argv){
  pid_t child;
  
  child = fork();
  if(child == 0)
    debugged_program(argv[1]); //argv[1] represents name of the program we want to trace
  else
    debugger(child);
  
  return 0;
}
```
According to above quotes, after calling `fork()` we may to set trace on child using PTRACE_TRACEME:

>PTRACE_TRACEME  
Indicate that this process is to be traced  by  its  parent. A process probably shouldn't make this request if its parent isn't expecting to trace it. (pid, addr, and data are ignored.) 
The PTRACE_TRACEME request is  used  only  by  the  tracee;  the remaining  requests are used only by the tracer.  In the following requests, pid specifies the thread ID of the  tracee  to  be acted  on.

Sounds good, let's code that:
```c
//executing the program we want to debug
int debugged_program(const char* programName){
  
  printf("Target started. Will run '%s'\n", programName);
  ptrace(PTRACE_TRACEME, 0, NULL, NULL);
  execl(programName, programName, NULL); //we are using execl instead of execve

  return 0;
}
```

Now we can trace our program! But how our parent process will know about any action?

>While  being  traced, the tracee will stop each time a signal is delivered, even if the signal is being ignored.  (An exception  is  SIGKILL, which  has  its usual effect.)  The tracer will be notified at its next call to waitpid(2) (or one of the related "wait"  system  calls);  that call  will  return a status value containing information that indicates the cause of the stop in the tracee.  While the tracee is stopped,  the tracer  can  use  various  ptrace  requests  to  inspect and modify the tracee.  The tracer then causes  the  tracee  to  continue,  optionally ignoring  the  delivered  signal (or even delivering a different signal instead).

Can you feel this power?! _Ok, ok, ok... but what is the waitpid(2)?_ I think at this point you know what you should do if you see something for the first time ;)  
I assumed that you did your homework so I will shortly sum all up. Whenever our _TREACEME_ change it's state, information about it will be sent to the parent of the process. Parent wait's for this signal using `waitpid` and for parent this is blocking operation (something like input). So in my understanting we can think of it as synchronous transmission. We now know everything to write function for our parent.

```c
//our main debugging function - our debugger
//looots of code was skipped
int debugger(pid_t childPID, functionList **head){
  int32_t wait_status, ret;

  puts("Debugger started!\n");

  waitpid(-1, &wait_status, 0);
  
  /*
  * WIFSTOPPED(status) 
  * returns  true  if the child process was stopped by delivery of a
  * signal; this is possible only if the call was  done  using  WUNTRACED 
  * or when the child is being traced (see ptrace(2)).
  */
  while(WIFSTOPPED(wait_status)){
    printf(">>");
    fgets(input, sizeof(int8_t)*INPUT_SIZE, stdin);
    ret = instruction_switch(instr, head, childPID, &break_head);
}
```
As long as we will receive _True_ from `WIFSTOPPED` we will perform `instruction_switch`. You may see in my repository what instruction switch does, but shortly it just makes the program dependent on the entered command, by using different version of `ptrace`. For example if user will input follwing line: **i r**, the program will be redirect to the function `print_registers(childPID)`:

```c
//printing actual state of the registers
void print_registers(pid_t childPID){
  ptrace(PTRACE_GETREGS, childPID, 0, &regs);
  printf("rip: 0x%08x\n"
      "rbp: 0x%08x\n"
      "rsp: 0x%08x\n"
      "rax: 0x%08x\n"
      "rcx: 0x%08x\n"
      "rdx: 0x%08x\n"
      "rsi: 0x%08x\n"
      "rdi: 0x%08x\n", regs.rip, regs.rbp, regs.rsp, regs.rax, regs.rcx, regs.rdx, regs.rsi, regs.rdi);
  
}
```
Here we get actual status of registers using `PTRACE_GETREGS` and `ptrace` and writing it to `regs` which is structure definied in `sys/user.h`[1].  
As you can see using different `enum __ptrace_request request` which is the first parameter of `ptrace` we can make program which we are debbuging to act as we want it!  
Full list of request _(as always)_ you may find in manual page. I will paste the most useful _(I think)_:

>PTRACE_PEEKTEXT, PTRACE_PEEKDATA  
Read a word at the address addr in the tracee's memory,  returning the word as the result of the ptrace() call.  Linux does not have separate  text  and  data  address  spaces,  so  these  two requests  are  currently  equivalent.  (data is ignored; but see NOTES.)  
>PTRACE_POKETEXT, PTRACE_POKEDATA  
Copy  the  word data to the address addr in the tracee's memory. As for PTRACE_PEEKTEXT and PTRACE_PEEKDATA, these  two  requests are currently equivalent.  
>PTRACE_GETREGS, PTRACE_GETFPREGS  
Copy the tracee's general-purpose or  floating-point  registers, respectively,   to   the   address  data  in  the  tracer.   See <sys/user.h> for information on the format of this data.   (addr is  ignored.)   Note that SPARC systems have the meaning of data and addr reversed; that is, data is ignored  and  the  registers are copied to the address addr.  PTRACE_GETREGS and PTRACE_GETFPREGS are not present on all architectures.  
> PTRACE_SETREGS, PTRACE_SETFPREGS  
Modify the tracee's general-purpose or floating-point registers, respectively,  from  the  address  data  in  the tracer. As for PTRACE_POKEUSER, some general-purpose register modifications may be disallowed.  (addr is ignored.)  Note that SPARC systems have the meaning of data and addr reversed; that is, data is  ignored and   the   registers   are   copied   from  the  address  addr. PTRACE_SETREGS and  PTRACE_SETFPREGS  are  not  present  on  all architectures.

I think it is everything for our first and fast introduction. If you can't wait for the next one or maybe you are just very curious all `lbdb debugger` code is already at my github!  
In link section you will find lots of useful links I found very useful writing my own debugger.

----
****
[1]. https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/host/x86_64-linux-glibc2.7-4.6/+/jb-dev/sysroot/usr/include/sys/user.h  
[2]. https://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/  
[3]. https://blog.tartanllama.xyz/writing-a-linux-debugger-setup/  
[4]. http://www.linuxjournal.com/article/6100?page=0,1



