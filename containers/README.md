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
The code is availabe in https://github.com/bennojoy/technologies_explained/blob/master/containers/code/bash_container.c

To illustrate this lets a look at some command outputs on a regular bash shell and a isolated/containerized bash shell:

####Normal Bash shell:
-----------------------


![Alt text](/containers/images/container_host.png "Arch")  
  


####Containerized  Bash shell:
------------------------------

![Alt text](/containers/images/container_cont.png "Cont")  
     

####Process Isolation
---------------------

If we look the normal bash shell output of the command 'pstree -p' we can see the entire process tree of the sytem, from the sytemd process which has the process id 1 to our current shell process which initiated the pstee command (pid 11734).
This is the default 'process namespace' of the system and any new process would be added to this namespace.

Now if we look at the pstree -p output from the containerized bash shell we see that we only two process pid 1, which is our bash shell and pid 13 the pstree process. 
This shell was created a new process namespace and this namespace has only these two processes, so this process cant see any other process running on the system. so basically this process is containerized and has no connections with other process.

Now let see how this is achieved, The magic of process isolation happens in the 'clone' system call, remember that 'clone' creates a new process in the system, if we pass a FLAG 'CLONE_NEWPID' to the clone call, the process would be created in a new process namespace isloated from default namespace.

Here is code snippet this make this happen.

```
clone:

child_pid = clone(childFunc,
                    child_stack + STACK_SIZE,   /* Points to start of
                                                   downwardly growing stack */
                    CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET| SIGCHLD, argv[1]);

here clone sytem call is passed 'childFunc' which is fucntion that would be called when child process is created and we also pass the CLONE_NEWPID Flag to detach from default namespace.

childFunc call:

printf("childFunc(): PID  = %ld\n", (long) getpid());
    printf("childFunc(): PPID = %ld\n", (long) getppid());

    char *mount_point = "/proc";

    if (mount_point != NULL) {
        if (mount("proc", mount_point, "proc", 0, NULL) == -1)
            errExit("mount");
        printf("Mounting procfs at %s\n", mount_point);
    }

    execlp("bash", "bash", (char *) NULL);


prints the process id's and executes the bash shell using the execlp system call in this new namespace.

```     

###Mount Isolation
------------------

The flag CLONE_NEWNS to the clone system call give the newly created process a diffrent mount namespace to the process. So for example in the child process above we mount /proc in the child process and we see that the contents are totally diffrent in the normal shell and continer shell.
So things mounted in the new process would be only visible to the new process.

###Network isolation
-------------------

Network isolation is got by passing the flag CLONE_NEWNET, with this flag a new network stack is added for the process, so if we look at the ifconfig output from the contianerized shell we see that there are no interfaces.
We can create virtual interfaces and add to this new network namespaces. Below is command to do that.

ip link add name veth0 type veth peer name veth1 netns <pid>

where pid is the child process, which in our case is '11781' . Look at the output on the containerized bash screenshot, it print the process id of the child from the parent process namespace. 
This command should be executed from the parent shell as the child shell doesnt not about 11781.

###Other Isolations
-------------------

Other isolations like hostname (UTS), ipc etc.. work pretty much the same way by specifying the FLAG. But i guess this shoudl give a brief understanding of how containers are made. The example bash continer script pretty much acts like an 'appliacation container'.





