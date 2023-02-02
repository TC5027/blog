# Nonblocking I/O (unix os)

A process references I/O streams (= stream of bytes) with descriptors or file descriptors.
These descriptors are created by system calls like ```socket```,... or inherited from parent process via ```fork```.

## Reminder
When calling a syscall like ```open("file.txt")``` the output is an int corresponding to a file descriptor. More precisely it corresponds to an index for the file descriptor table specific to the process. 
Index 0, 1 and 2 are reserved for stdin, stdout and stderr.

At the kernel level we have

Process table         Open file table             inode or vnode table       
-----                 -----                       -----
pid : ...         |-> file status flags : ... |-> file type : ...
FD table : ---    |   file offset : ...       |   function pointers (for read, write...)
                  |   vnode pointer ----------|   inode ------------|
           ---    |   refcount : ...              refcount : ...    |
                  |   -----                       -----             |
           ---    |   file status flags : ...     file type : ...   |
                  |   ...                         ...               |
           ---    |   -----                       -----             |
             -----|                                                 |
           ---                                                      |
-----                                                               |
pid : ...                                                           |
...                                                                 |
-----                                                               |
                                                                    |
                                                                    |
And at filesystem level we have                                     |
                                                                    |
Inode table                                                         |
-----                                                               |
metadata         <--------------------------------------------------|
block pointers (as files are stored as blocks)
-----

## Nonblocking

By default, read on any descriptor blocks if there's no data available. Any descriptor can be put in the nonblocking mode, in which case an I/O system call on that descriptor will return immediately one of:
* an error
* a partial count
* the entire result

We say that a descriptor is ready if a process can perform an I/O operation on it without blocking (= error returned)

There are 2 triggering methods for interrupts to act on the readiness status of a descriptor:
* Edge-triggered interrupt : signaled by a level transition
   |-----|
   |     |
---|     |---
   ^
* Level-triggered interrupt : hold while at a specific level
   |-----|
   |     |
---|     |---
    ^^^^^

In the case of a web server we have to multiplex I/O on multiple file descriptors and **epoll** helps us to do so efficiently.

epoll is not a system call but a kernel data structure. It is often hidden behind "async" in programming languages like Rust.

The epoll instance is created with the ```epoll_create``` system call which returns a file descriptor to the epoll instance. The calling process can use this file descriptor to add, remove or modify other file descriptors it wants to monitor for I/O  to the epoll instance.
To add, remove or modify a file descriptor to the epoll instance we use ```epoll_ctl```. Then to be notified of events that happened on the epoll set of an epoll instance we use ```epoll_wait``` which blocks until any of the descriptors being monitored becomes ready for I/O.

I wrote a Rust version of a simple HTTP server using (level triggered) epoll which is available [here](https://github.com/TC5027/rust_stuff/tree/master/epoll_server).

## Testing

To evaluate it we can use ```siege```, an HTTP stress tester.
I run ```siege -c250 -r100 http://127.0.0.1:8000``` with server running in another terminal and got the following result
```
{	"transactions":			       25000,
	"availability":			      100.00,
	"elapsed_time":			        1.93,
	"data_transferred":		        0.12,
	"response_time":		        0.01,
	"transaction_rate":		    12953.37,
	"throughput":			        0.06,
	"concurrency":			      176.39,
	"successful_transactions":	       25000,
	"failed_transactions":		           0,
	"longest_transaction":		        1.13,
	"shortest_transaction":		        0.00
}
```
The availability is quite satisfying (the server is clearly not the most complicated ever). I also inspected cpu usage with top and the server does not exceed 30% on my machine.

## References 
* https://en.wikipedia.org/wiki/File_descriptor
* https://www.zupzup.org/epoll-with-rust/
* https://man7.org/linux/man-pages/man7/epoll.7.html
* https://linux.die.net/man/1/siege


