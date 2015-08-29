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
That is defnietly possible and  we could rewrite the application in such a way that it spawns threads/copies equal to the number of cpu's it has and give each copy a share of the numbers to cruch. This is very widely used in current applications.

### Distributed Processing.

Now lets assume we have a datacenter with 100 machines and each machine having 4 cores and most of the times these cpu's are used on 10%. What if we could spawn our factorial application for each of this cpu across all our servers/nodes ? That would be awesome we would reduce the time to caclulate drastically and if we want to scale just add more servers to the datacenter. This is exactly what Apache Mesos allows us to do.

### Apache Mesos

If we look at our first factorial program we see that inorder to run it the os kernel does stuffs like allocating cpu/memory etc..with Mesos we dont have a single server to run our application instead the whole datacenter acts as one server and mesos handles sceduling cpu/memory/disks/ports etc... for our application, apart from scheduling mesos also has capabilitiies like fault tolerance, resouce isolation etc...

   
#### Architecture.

![Alt text](/mesos/mesos_ansible/images/mesos_arch.png "Arch")





The Apache Mesos itself  consists of the Mesos Master and Mesos slave processes and the applications that use mesos to run on the clsuter are called as Frameworks.


####Mesos Master: 

    The master is a process which runs on a node in the cluster and orchestrates the running of tasks on slaves by receiving resource offers(cpu/mem/port/disk) from slaves and offering those resources to registered frameworks(Factorial).

####Mesos Slave  

    The slave is a process which runs on a node in the cluster and offers up resources available on that node to the Mesos Master. The slave also takes schedule requests from the master and invokes an executor to launch a task (factorial function).

###FrameWorks.
--------------


Inorder for our factorial program to work in a distributed way we would have to split the application into two components and lets call this two components combined as 'factorial framework' so that we are inline with the Mesos terminology.

####Scheduler 

The first part of the factorial framework would be the schedular, The schedular handles many fuctions like 

    - Interface to get the list of numbers for which factorial should be calculated.
    - Accept/Reject offers(cpu/mem)  from the mesos master.
    - Prepare the tasks with required cpu/mem for the executors to execute.
    - Handle failures of tasks in executors etc..

Lets look at some code snippets and explanation of how the schedular would be implemented. We use python as the framework langugage.


```

#The complete code is available in the mesos ansible role

#The main function 
if __name__ == "__main__":

#Instantiate the builtin ExecutorInfo Class
    executor = mesos_pb2.ExecutorInfo()
    executor.executor_id.value = "default"

#This is where we tell the  name of the binary which should be invoked in the slave nodes, it is 'factorial_executor' in this case and this should be availabe in the slave nodes, so we can have an hdfs filesystem so that it is available on all slave nodes or use ansible to copy this executable to all slave node.
    executor.command.value = "factorial_executor"
    executor.name = "Factorial Executor (Python)"

#Instanitate the Framework 
    framework = mesos_pb2.FrameworkInfo()
    framework.user = "" # Have Mesos fill in the current user.
    framework.name = "Factorial Framework (Python)"

#Fill up connection details  to the mesos master and tell which is our schedular implementation class 'FactorialScheduler'.
    driver = mesos.native.MesosSchedulerDriver(
            FactorialScheduler(implicitAcknowledgements, executor),
            framework,
            "mesos_master:5050",
            implicitAcknowledgements)

#Run the scheduler.
    status = 0 if driver.run() == mesos_pb2.DRIVER_STOPPED else 1

# Ensure that the driver process terminates.
    driver.stop();




#
#The actual implementation of the factorial schedular, which implements the schedular interface
# 

#Initializing some variables we would be using.

TOTAL_TASKS = 2
FACTS = [ '5', '3']  #The numbers for which factorial should be calculated
TASK_CPUS = 1        #The number if cpus to be allocated for each task (each number)
TASK_MEM = 10        #The amount of memory to allocated for each task (each number)

#The interface implementation.

class FactorialScheduler(mesos.interface.Scheduler):
    def __init__(self, implicitAcknowledgements, executor):
        self.implicitAcknowledgements = implicitAcknowledgements
        self.executor = executor
        self.taskData = {}
        self.tasksLaunched = 0
        self.tasksFinished = 0
        self.messagesSent = 0
        self.messagesReceived = 0

#This function is called when we register ourselves with the mesos master as a framework    
    def registered(self, driver, frameworkId, masterInfo):
        print "Registered with framework ID %s" % frameworkId.value

#This fucntion is called when we get a resource offer fromt the mesos master.
    def resourceOffers(self, driver, offers):
        for offer in offers:
#The first offer we got            
            tasks = []
            offerCpus = 0
            offerMem = 0
            for resource in offer.resources:
                if resource.name == "cpus":
#The number if cpu's in this offer
                    offerCpus += resource.scalar.value
#The amount of memory in this offer
                elif resource.name == "mem":
                    offerMem += resource.scalar.value

            print "Received offer %s with cpus: %s and mem: %s" \
                  % (offer.id.value, offerCpus, offerMem)

            remainingCpus = offerCpus
            remainingMem = offerMem
            
            while self.tasksLaunched < TOTAL_TASKS and \
                  remainingCpus >= TASK_CPUS and \
                  remainingMem >= TASK_MEM:
                tid = self.tasksLaunched
                self.tasksLaunched += 1

                print "Launching task %d using offer %s" \
                      % (tid, offer.id.value)

#Create a task with one cpu and 10 mb of ram and have the data populated with the number for which factorial should be calculated.
 
                task = mesos_pb2.TaskInfo()
                task.task_id.value = str(tid)
                task.slave_id.value = offer.slave_id.value
                task.name = "task %d" % tid
                task.data = FACTS[self.tasksLaunched - 1]
                task.executor.MergeFrom(self.executor)

                cpus = task.resources.add()
                cpus.name = "cpus"
                cpus.type = mesos_pb2.Value.SCALAR
                cpus.scalar.value = TASK_CPUS

                mem = task.resources.add()
                mem.name = "mem"
                mem.type = mesos_pb2.Value.SCALAR
                mem.scalar.value = TASK_MEM

                tasks.append(task)
                self.taskData[task.task_id.value] = (
                    offer.slave_id, task.executor.executor_id)

                remainingCpus -= TASK_CPUS
                remainingMem -= TASK_MEM

            operation = mesos_pb2.Offer.Operation()
            operation.type = mesos_pb2.Offer.Operation.LAUNCH
            operation.launch.task_infos.extend(tasks)

#Launch the task

            driver.acceptOffers([offer.id], [operation])

#This fucntion is called whenever the executor updates its status , it can also be invoked by the master incase task is lost or terminated etc..

    def statusUpdate(self, driver, update):
        print "Task %s is in state %s and data %s" % \
            (update.task_id.value, mesos_pb2.TaskState.Name(update.state), update.data)

        if update.state == mesos_pb2.TASK_FINISHED:
            self.tasksFinished += 1

#Here we print the factorial as the executor updates it status as FINISHED and also adds the result in the update data.

            print update.data

#If we got factorial for all data , stop this framework.
            if self.tasksFinished == TOTAL_TASKS:
                print "All tasks done"
                driver.stop()
            slave_id, executor_id = self.taskData[update.task_id.value]

#Abort in case any tasks failed or lost etc..

        if update.state == mesos_pb2.TASK_LOST or \
           update.state == mesos_pb2.TASK_KILLED or \
           update.state == mesos_pb2.TASK_FAILED:
            print "Aborting because task %s is in unexpected state %s with message '%s'" \
                % (update.task_id.value, mesos_pb2.TaskState.Name(update.state), update.message)
            driver.abort()

        # Explicitly acknowledge the update if implicit acknowledgements
        # are not being used.
        if not self.implicitAcknowledgements:
            driver.acknowledgeStatusUpdate(update)

```







####Executor

Executor is where we implement the factorial logic and also handle things like launching the task, updating the status of this execution to the schedular and also communication with the schedular.

Lets have a look at the code snippets of the factorial executor.

```

# The main function
if __name__ == "__main__":
    print "Starting executor"

#Specify the fucntion to be called when this executor is invoked.

    driver = mesos.native.MesosExecutorDriver(FactorialExecutor())
    sys.exit(0 if driver.run() == mesos_pb2.DRIVER_STOPPED else 1)

   


#The Executor implementation.

class FactorialExecutor(mesos.interface.Executor):

#This fucntion would be called when a task is launched, so we add our code here.
    def launchTask(self, driver, task):
        # Create a thread to run the task. Tasks should always be run in new
        # threads or processes, rather than inside launchTask itself.
        def run_task():
            print "Running task %s" % task.task_id.value

#Update our factorial schedular that the task is started. In the factorial:Schedular code statusUpdate(self, driver, update) fucntion is invoked. 
            update = mesos_pb2.TaskStatus()
            update.task_id.value = task.task_id.value
            update.data = task.data
            update.state = mesos_pb2.TASK_RUNNING
            driver.sendStatusUpdate(update)

# This is where one would perform the factorial calculation.
            fact = 1
            for i in range(1, int(task.data) + 1):
                fact = fact * i

            print "Sending status update..."

#Finished now let us update the schedular with the result

            update = mesos_pb2.TaskStatus()
            update.task_id.value = task.task_id.value
            update.state = mesos_pb2.TASK_FINISHED
            update.data = "The factorial for %s is %s" % (task.data, fact)
            driver.sendStatusUpdate(update)
            print "Sent status update"

        thread = threading.Thread(target=run_task)
        thread.start()



```
