## Introduction to Containers
-----------------------------

Containers is the way Linux  kernel  isolate's system resources like processes, filesystem mounts, networks etc... so that they can work independently without stepping on each other. As an example lets say we need multiple  http server application running and they use port 80 and we only have one machine and we dont want to use virtualization since they are pretty heavy, these is were containers help us.

###Namespaces
-------------

The fundamental building blocks of containers are the kernel implementation of namespaces, before diving into namespaces lets have a look at how an application is started in Linux. The usual process is to login to linux system via ssh which should give us a shell and the start the application via typing the application/binary name in the shell.

    $./a.out

The first thing shell does is makes a fork/clone system call which creates a new process (child)  which is an exact copy of the parent ( in this case the shell) and  all the open file descriptors etc... are shared with the child. 

Then immediately the child process  makes a system call 'execve' with arguments './a.out'. The execve system call then loads this a.out' binary from the disk and replaces this new process address spaces with the new a.out's binary and is executed.

The parent process(shell) waits for the child process to complete using the wait system call and when finished the prompt is give back to the user.

### Issues with traditional clone/process
-----------------------------------------

In the section above we saw how a new process is created by the kernel via fork/clone, There are a few drawbacks with this approach with respect to process isolation. Lets have a look a few of them.

```

Process Isolation:
   If the a.out binary had call like system(kill, -9, 1) and it was run as root, it would shutdown the whole system. 

Filesystem/mount Isolation:
   What if my application needs mounts/filesytems that are not visible to any other process.

Network Isolation:
  My application needs port 80 to work and another application is using it.

Hostname Isolation
  The application needs it's own hostname to identify itself but it should not be the same as the current hostname

IPC isloation
  The application uses ipc like pipes/shared mem  etc... and does want this to be available to other process.

User isolation
  The application needs it's own users like root etc.. and they should not conflict with the existing sytem users.

```


### NamesSpaces implementation
------------------------------

All the above stated problems are solved by namespaces, Lets see how it is done with a simple example program. This program start a bash shell which is totally isolated from other processes and isolated from system network and filesytems isolation.

To illustrate this lets a look at some command outputs on a regular bash shell and a isolated/containerized bash shell:

Normal Bash shell:

![Alt text](/containers/images/container_host.png "Arch")  
  

Containerized  Bash shell:

![Alt text](/containers/images/container_cont.png "Cont")  
      
