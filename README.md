## All Pseudocode and Important Functions: COMP 310 Midterm

Notes for Fall 2019 Midterm for COMP 310 at McGill. All code is written as Java-like or C-like pseudocode. 

[TOC]

## 03 - Processes 

#### Forking (C code)

``` c
int main () {
    pid_t pid;
    pid = fork();
    if (pid < 0) { //Error
    } else if (pid == 0) { //Child
        // Overlays address space w/ a new program. Doesn't return unless something went wrong
        execlp(...); 
    } else { //Parent
        wait(NULL);
    }
}
```

## 03 - 1 - More on Processes

By default (i.e. at process creation), file descriptor 0 points to standard input, 1 to standard output, and 2 to standard error. 

#### Reading, writing, and opening files

```C
// read() attempts to read up to count bytes from file descriptor fd into the buffer starting at buf.

ssize_t read(int fd, void *buf, size_t count);

//write() writes up to count bytes from the buffer starting at buf to the file referred to by the file descriptor fd.

ssize_t write(int fd, const void *buf, size_t count);

// The open() system call opens the file specified by pathname.

int fd = open(const char *pathname, int flags, mode_t mode);
// mode is optional
//Flags include O_RDONLY, O_WRONLY, O_RDWR

//close() closes a file descriptor, so that it no longer refers to any file and may be reused. 
int close(int fd);
```

#### Manipulating file descriptors

```c
//pipe() creates a pipe, a unidirectional data channel that can be used for interprocess communication. The array pipefd is used to return two file descriptors referring to the ends of the pipe. pipefd[0] refers to the read end of the pipe. pipefd[1] refers to the write end of the pipe

int pipe(int pipefd[2]);

// The dup() system call creates a copy of the file descriptor oldfd, using the lowest-numbered unused file descriptor for the new descriptor.

int dup(int oldfd);

//The dup2() system call performs the same task as dup(), but instead of using the lowest-numbered unused file descriptor, it uses the file descriptor number specified in newfd. 

int dup2(int oldfd, int newfd);

```

#### Unidirectional pipe example

The parent reads from its child; i.e. the output of child goes into input of parent

``` C
int fd[2];
pipe(fd);
int pid = fork();

if (pid == 0) { //Child
    close(1); //Makes sure that this is closed so it can be used
    dup(f[1])
} else {
    close(0);
    dup(f[0]);
}
```

#### Bidirectional pipe example

Bidirectional piping so that standard output of one process is the standard input of the other, and vice versa. 

![image-20191028172430551](/Users/jiawen/Library/Application Support/typora-user-images/image-20191028172430551.png)

```c
int forward[2], back[2];
pipe(forward);
pipe(back);
int pid = fork() {
    if (pid == 0) {
        close(0);
        dup(forward[0]);
        close(1);
        dup(back[1]);
    } else {
        close(1);
        dup(forward[1]);
        close(0);
        dup(back[0]);
    }
}
```

#### Signal Handling

``` c
int main () {
    signal(SIGINT, interruptHandler);
    while(1) {}
    return 0;
}

void interruptHandler(int sig) {
    printf("Caught Interrupt %i\n", sig);
}
```



## 04 - Threads

### Pthreads in C

#### Pthread important functions

```c
#include <pthread.h>

//Return 0 on success
int pthread_create(phtread_t * thread, const pthread_attr_t * attr, void *(*start)(void*), void *arg);

pthread_exit(void * retval); //exits the thread
pthread_t pthread_self(void); //returns ID of calling thread
pthread_equal(pthread_t p1, pthread_t t2); // returns non-zero if two threads are equal

//Returns 0 on success
int pthread_join(pthread_t thread, void **retval);

//Failing to join a created thread can create zombie threads
    
```

#### Pthread usage example

```c
#include <phread.h>
static void * threadFunc(void *arg) {
    char *s = (char *) arg; //CAST VOID POINTER FIRST!!
    // .. Do something ..
    return;
}
int main (int argc, char *argv[]) {
    pthread_t t1;
    void *result;
    int s;
    s = pthread_create(&t1, NULL, threadFunc, "Hello world\n");
    if (s != NULL) { 
        //Something went wrong..
    }
    s = pthread_join(t1, &result);
    if (s != 0) {
        // You get it
    }
    //do something with result
    exit(EXIT_SUCCESS);
}
```

#### The fork-join model

Pseudocode of a recursive algorithm generally used in Java:

```pseudocode
Task(problem)
	if problem small enough
		solve it directly
	else
		subtask1 = fork(new Tast(subset of problem))
		subtask2 = fork(new Tast(subset of problem))	
        
      result1 = join(subtask1)
      result2 = join(subtask2)
        
      return combined results

```

#### Protecting shared variables in threads

```c
int pthread_mutex_lock(pthread_mutex_t * mutex);
int pthread_mutex_unlock(pthread_mutex_t * mutex);


//Example: if example threadFunc instead had to manipulate a global variable....
static int glob = 0;
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;

static void * threadFunc(void * arg) {
    int loops = *((int *) arg);
    int loc, j, s;
    for (j = 0; i < loops; j++) {
        pthread_mutex_lock(&mtx); 
        loc = glob;
        loc++;
        glob = loc;
        pthread_mutex_unlock(&mtx); 
    }
}
```



## 05 - Synchronization

#### 4 requirements for a viable solution to synchronization

1. No two processes should be in their critical sections **simultaneously**
2. No process **not in its critical section** should prevent another process from entering its critical section
3. No process should **have to wait forever** to get into its critical section; that is, processes should always expect to make progress
4. No **assumptions** should be made about the number of CPU cores or their clocks speeds

### Road to a Solution: Mutual Exclusion

#### Attempt #1

Single key shared between two processes

```c
// Process 0
while (turn != 0); //wait
critical_section();
turn = 1;
```

```c
// Process 1
while (turn != 1); //wait
critical_section();
turn = 0;
```

Reason why this doesn't work: 2 and 3 (strict alteration and spin lock/busy waiting)

#### Attempt #2

Each process has their own key; each process only enters its critical section when it thinks that the other process is not attempting to do so.

```c
// Process 0
flag[0] = false; //init

while (flag[1]);
flag[0] = true;
critical_section();
flag[0] = false;
```

```c
// Process 1
flag[0] = false; //init

while (flag[0]);
flag[1] = true;
critical_section();
flag[1] = false;
```

Fails by rule 1: context switches can make it so that both processes are in their critical sections at the same time. 

The following sequence of events might end with both processes in their critical sections.

1. Process 0 inits, and runs its line 4 `while` loop
2. Context switch to Process 1, where it also inits and runs its line 4 while loop
3. From Proc 1's point of view, flag[0] is false so it enters its critical section. 
4. After running Process 1's line 5, another context switch.
5. Process 0 continues on, running its line 5. Now both progress, and now both processes are in their CS.



#### Attempt #3

Like attempt #2 but to solve the above-mentioned problems we switched lines 5 and 6.

```c
// Process 0
flag[0] = true;
while (flag[1]);
critical_section();
flag[0] = false;
```

```c
// Process 1
flag[1] = true;
while (flag[0]);
critical_section();
flag[1] = false;
```

Violates 3; can result in **deadlock** where no process progresses. Imagine context switch after line 2. Then both flag[0] and flag[1] are true; neither progress. 

#### Attempt #4

Like attempt #3 but in the while loop we attempt to solve the problem by waiting a random amount of time as a tiebreaker, so whoever has the longer random wait "yields" to the toher process. 

```c
// Process 0
flag[0] = true;
while (flag[1]) {
   flag[0] = false;
   random_delay();
   flag[0] = true;
}
critical_section();
flag[0] = false;
```

```c
// Process 1
flag[1] = true;
while (flag[0]) {
   flag[1] = false;
   random_delay();
   flag[1] = true;
}
critical_section();
flag[1] = false;
```

Suffers the same problem as Attempt #3 if random_delay() happens to give the same number.

This results in a **livelock** where both processes are making progress (in the while loops) but just not where we *want* to make progress (the while loops aren't doing any real work pertaining to the critical sections).

#### Peterson's Algorithm

Peterson's Algorithm is the first non-semaphore solution we've seen that actually works. The idea is to use the same idea as attempt 3 and 4 but bring back the `turn` variable from attempt 1 as a tiebreaker for when the two processes are in the same state. 

*Note that Peterson's Algorithm only works for two processes.*

```c
//Process 0
flag[0] = true;
turn = 1;
while (flag[1] && turn == 1);
critical_section();
flag[0] = false;
remainder_of_program();
```

```c
//Process 1
flag[1] = true;
turn = 0;
while (flag[0] && turn == 0);
critical_section();
flag[1] = false;
remainder_of_program();
```



### Monitors

#### Implementation of a Semaphore using a Monitor

```Java
SemaphoreMonitor {
   int count;
   ConditionalVar block;
   
   void sem_wait() {
      count--;
      if (count < 0) {
         block.wait();
      }
   }
   
   void sem_signal() {
      count++;
      block.signal(); //don't need the if because semaphore will simply do nothing if nothing is waiting
   }
   
   void init() {
      count = 0;
   }
   
}
```

#### Producer Consumer Monitor

[https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem](https://en.wikipedia.org/wiki/Producer–consumer_problem)

```java
Monitor ProducerConsumer {
   int itemCount = 0;
   int BUFFER_SIZE = 100;
   ConditionVar full;
   ConditionVar empty;
   
   void add(item) {
      if (itemCount >= BUFFER_SIZE) {
         full.wait();
      }
      pushItem(item);
      itemCount++;
      if (itemCount == 1)
      	empty.signal();
   }
   
   void remove() {
      if (itemCount <= 0) {
         empty.wait();
      }
      itemCount--;
      item = popItem();
      
      if (itemCount == BUFFER_SIZE - 1)
         full.signal();
      
      return item;
   }
}


//Usage
void Producer() {
   item = produce_item();
   ProducerConsumer.add(item)
}

void Consumer() {
   item = ProducerConsumer.remove(item);
   consume_item(item);
}
```

#### Producer-Consumer with Semaphores

```c
int BUFFER_SIZE; //Capacity of buffer
sem filled = 0; //Number of spots in buffer filled
sem empty = BUFFER_SIZE; //Number of spots in buffer empty
sem mutex; //used to protect the queue itself

Producer {
   int item;
   while (true) {
      item = produce_item();
      wait(empty); //Make sure there's empty space left in the buffer
      wait(mutex);
      insert_item(item);
     	signal(mutex);
      signal(filled); //Filled a spot
   }
}

Consumer {
   int item;
   while (true) {
      wait(filled); // Make sure there's something to consume
      wait(mutex);
     	item = remove_item();
      signal(mutex);
      signal(empty); //Opened up a spot
      consume_item(item);
   }
}
```

#### Implementing a counting semaphore via 2 binary semaphores

At most N threads are allowed to access a certain resource. We want to keep track of the number of processes currently in its critical section with `count`, and make sure that we are not exceeding this number. In addition, since `count` is a shared variable, we must protect it with a binary semaphore as well. 

This yields the following...

(((update me)))

## 06 - Deadlocks

![image](https://i.imgur.com/08fgfs0.png "Deadlocks table")

