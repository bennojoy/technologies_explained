## Apache Mesos



What is Mesos? (As in the mesos website)

A distributed systems kernel

Mesos is built using the same principles as the Linux kernel, only at a different level of abstraction. The Mesos kernel runs on every machine and provides applications (e.g., Hadoop, Spark, Kafka, Elastic Search) with API’s for resource management and scheduling across entire datacenter and cloud environments.



###An Example

I think an example application would enable us to understand the background concepts a bit more. So let's say we have need an application which calcluates and prints factorial of a 'list' of numbers.


Lets look at a psuedo code which implements this factorial application and can be run on any machine.


```

facts = [5, 7, 10, 70, 30]

for i in facts:
    print factorial(1)

```

Now lets assume we run this application in our standard linux shell, These are the high level things the kernel would do for us to run this appplication.

```

1) Fork a new process. (The 'memory' manager in kernel allocates some free pages for the applications code, data, stack segements etc..)

2) The loader loads the application in to the memory and adds it into the systems process list so that it can be scheduled and run.

3) The scheduler based on priority etc.. grabs the process from the list and allocates a 'cpu' for it to run.


```

Now lets have a look at the application run itself. 

- It first grabs the first number (5) from the list and pass it on the function factorial, which then calculates the factorial and prints it.

- Then when the fist result is printed the application grabs the second number from the list and passes it on to the factorial function and so on.


### Serial processing

The processing of these factorials are serial and can take significant time if there are millions of number to be calculated. So what can we do to speed things up.

### Parallel Processing

 we know that a process needs a cpu to run, now lets assume we have a 4 cpu/core machine, what if we could have 4 copies of this application and each get 1/4th of the list. 
That is defnietly possible we could rewrite the application in such a way that it spawns threads/copies equal to the number of copus it has and give each copy the shared number to cruch. This is very widely used in current applications and lets call this 'Parallel Processing'.

 


