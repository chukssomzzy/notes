
# A Simple Echo Server & Client 

1. The client read line from it standard input and writes the line to the server. 
2. The server reads the line from its network input, writes the line back to the client. 
3. The Client reads the line from its network input, then write it to standard output and wait for another input from standard input to repeat the process from 1 

![[Simple Echo Client & Server.canvas|Simple Echo Client & Server]]

## Boundary Conditions To Examine 
1. What happens when the client and server are started. 
2. What happens when the server terminates normally.
3. What happens to the client when the server process terminates prematurely  

## TCP Echo Server Main Function 
The Socket function is called and it returns the socket descriptor, The socket descriptor is binded to address specified by `INADDR_ANY` which means the server would accept any connection destined on any local interface on the machine incase where the server is address is multihomed. The port assigned is 9877 to avoid conflit with the reserved address space 1023 and greater than 5000 (avoid conflict with ephemeral port used by many Berkeley derived implementation) also less than 49152 (avoid conflict with the correct range of ephemeral ports). The process then call `listen` to convert the socket descriptor to a listening socket. The `accept` is called which block till a client connection is completed. 

## Concurrent Server 
The server calls `fork` to spawns a child process and assigns the handling of the child connection to the child process. 

## Normal Startup 
When the server is started it call `socket`, `bind`, `listen` and blocks on the call to `accept`. When `netstat -a` is ran the server process is show to bind on a wildcard address on port 9877 and receives connection from any interface and port number. 
When the client process is ran with the local address as argument the it calls `socket` then `connect` which performs the 3-way handshake. After the server accepts the connection it forks a child process to deal with the client while closing the connection socket and going into blocking on accept to receive new connection. 
The client blocks on fget while the server child process blocks on read this leaves us with three sleeping processes. Now if we run `netstat -a` we see 
``` 
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:9877            0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:ssh             0.0.0.0:*               LISTEN
tcp        0      0 localhost:54148         localhost:9877          ESTABLISHED
tcp        0      0 localhost:9877          localhost:54148         ESTABLISHED
```

Two established socket connection, one for the child server process and client process and a Listen socket for the parent server process. 

```bash 
ps -t pts/6 -o pid,ppid,tty,stat,args,wchan
```

This command shows the process relationships the -t select the terminal the process is attached to and -o select the information that would be included in the output. 

## Normal Termination 
We terminate the client connection by typing `ctrl-d` . If we checked the state of the connection with `netstat -a | grep 9877`, it shows that the client has entered `FIN_WAIT2` state and server child process in `CLOSE_WAIT` state. 

```
tcp        0      0 0.0.0.0:9877            0.0.0.0:*               LISTEN
tcp        0      0 localhost:9877          localhost:54222         CLOSE_WAIT
tcp        0      0 localhost:54222         localhost:9877          FIN_WAIT2
```

While in the `CLOSE_WAIT` the server child process becomes a zombie process. 
### Steps in the Normal Termination of the client and server 

1. The client receives a "end of file EOF" this causes `fget` to return null causing `str_cli` to return to main. 
2. The main function calls `exit(0)` which causes the kernel to closes all file descriptors connected to the terminating process.
3. The closing of file descriptor causes a `FIN` segment to be sent from the client to the server and the server responds with a `ACK` segment this completes the first part of TCP closing sequence. The client enters a state FIN_WAIT2 and server enters CLOSE_WAIT. 
4. Once the server receives the FIN segment, it blocks on the call to readline and returns 0 to main, this causes main to call exit and closing of file descriptors take place. 
5. The server send a `FIN` segment to the the client which responds with a `ACK` terminating the TCP connection. The client enters a TIME_WAIT to ensure the connection is properly terminated. 
6. The server parent receives a `SIGCHLD` and child enters a zombie state till the parents terminates.

## POSIX Signal Handling

A signal is a notification to process that something has occured. Signal is also called a software interrupt. Signal occurs asynchronously so the process doesn't know when a signal would occur. 

### Signals can be sent 
1. By one process to another process 
2. By kernel to a process 

Every signal has an ISR (interrupt service routine) or a signal handler that is called when a signal is received and can be set by calling the `sigaction` function, this is known as the disposition of the signal.

### Choices for disposition (handler) 

1. The setting of a function to be called when a signal occurs is signal handling. While the action is called catching a signal. Some signal disposition cannot be set like `SIGSTOP` and `SIGKILL`. A handler is expected to return nothing and accept an integer which is the signal no. which leaves it prototype as 👇
```C 
void handler(int signo);
```

2. we can ignore a signal by setting it disposition to `SIG_IGN`, the two signals `SIGSTOP` and `SIGKILL` cannot be ignored.
3. The default signal handling can be used by setting the disposition to `SIG_DFL`. Normally the default disposition of a process receiving a signal is to terminate while some create a core image of the process in the current working directory. The default disposition of some signal is to be ignored like `SIGCHLD` and `SIGURG`. 

## `signal`  function

The POSIX way to establish disposition is by calling `sigaction` function which takes a struct and this can get complicated. A function called to set disposition and hides the implementation and takes 2 argument the signal name and handler function pointer or a pointer to a constant like `SIG_IGN` or `SIG_DFL` is `signal`, but `signal` predates posix. Hence different implements provides different signal semantics when it is called. The solution is to create a wrapper around `sigaction` that provides our desired signal semantics. 

```C 
# include "unp.h"

Sigfunc *Signal(int signo, Sigfunc *func)
{
	struct sigaction act, oact;

	act.sa_handler = func;
	sigemptyset(&act.sa_mask);
	act.sa_flags = 0;
	if (signo == SIGALRM)
	{

# ifdef SA_INTERRUPT
		act.sa_flags |= SA_INTERRUPT;
# endif
	} else
	{

# ifdef SA_RESTART
		act.sa_flags |= SA_RESTART;
# endif
	}
	if (sigaction(signo, &act, &oact) < 0)
		return (SIG_ERR);
	return (oact.sa_handler);
}
```



The normal function prototype for signal is 👇 
```C 
void (*signal (int, void (*)(int)) (int));
```

Typedef of `Sigfunc` uncomplicate the prototype 

```C 
typedef void Sigfunc(int);
```

We first set the `sa_handler` of the struct to the `Sigfunc` argument. Posix defines `sig_mask` which is used to define additional signal that would be blocked while the signal is being handled. We set it to an empty set with `sigemptyset`, so only the handled signal is blocked while it handler is executing. 
Any system call that is interrupted by a signal has to restarted. In the Signal wrapper we check if the received signal is `SIGALRM` then set the flag to `SA_INTERRUPT` because SIGALRM signal represent the timeout of a system call. And if it not a `SIGALRM` we restart the interrupted system call by setting the flag to `SA_RESTART`. Because some system does restart of interrupt signal automatically we have to make sure the `SA_RESTART` const is defined before setting them. 

after the call to `sigaction` with the signal number, new and old structure. If successful we return the `sig_handler` of the old structure else we return an `SIG_ERR`.

## POSIX Signal Semantics 

1. Once a signal handler is installed it remain (older implementation of POSIX removes the handler each time the signal is received) 
2. While the handler for a signal is executing the signal is blocked and also any other signal passed to `sig_mask` when the handler is installed is also blocked. 
3. If a signal is delivered one or more times after it is blocked it is only delivered one more time after the signal is unblocked and signals are never queued. 
4. It is possible to selectively block and unblock a set of signals using `sigprocmask` function. This helps us to protect a region of code preventing certain signals for being caught while a area of code is executing. 
## Handling `SIGCHLD` Signal 
The purpose of zombie state is to maintain process information about the terminated child for the parent to fetch at a later time. If a parent process terminates while still having some children in zombie state the `init` inherits the zombie children and cleans them up. The command section of a zombie child is usually represented as `<defunc>` on some UNIX implementation.

The `Signal` function is called only once after listen and before the fork of the first child. 

```C 
Signal(SIGCHLD, sig_chld)
```

The `sig_chld` function waits the terminating child, thereby causing the kernel to release any allocated resources fir that child process. 

```C 
/**
 * sig_chld - wait for a child to terminate
 * @signo: signal received
 */

void sig_chld(int __attribute__((unused)) signo)
{
	pid_t pid;
	int stat;

	pid = wait(&stat);
	printf("Child %i terminated", pid);
}
```

> Calling an IO function inside a handler is generally not recommend.

### Sequence Of Steps 
1. The client receives an EOF, terminates and sends `FIN` to server and server responds with an `ACK`. 
2. The server delivers an EOF to child process readline call, terminating child. 
3. `SIGCHLD` signal is sent to parent which is handled by `sigchld` that wait the child, returning the status and pid of the terminated child calling printf before return. 
4. When the child receives `SIGCHLD` signal it blocks in a slow system call to accept. This terminates the system call and on some implementation of signal the system call is not restarted by setting `SA_RESTART` flag on `sa_flag` there by accept returns `EINTR` to parent then terminates if not handled.

## Handling interrupted Signals.
Slow system calls are blocking calls that can block forever. In other to handle interrupt to accept I modified my Accept wrapper function to use a while loop that only terminates if the `errno` is not `EINTR`. 

```C 
# include "unp.h"


/**
 * Accept - wrapper around the accept function
 * @listenfd: the filedescriptor to check for cnnected client
 * @servaddr: address to listen on
 * @addrlen: the len of the addr to listen on
 * Return: the filedescriptor of the socket that wants to connect
 */

int Accept(int listenfd, struct sockaddr *servaddr, socklen_t *addrlen)
{
	int fd;

	while ((fd = accept(listenfd, (struct sockaddr *) servaddr, addrlen)) < 0)
		if (errno == EINTR)
		{
			continue;
		} else
		{
			perror("accept error");
			exit(EXIT_FAILURE);
		}
	return (fd);
}
```

Recalling of system calls are fine for function such as `read`, `write`, `select`, `open`. But when connect function is interrupted by a caught signal. Recalling `connect` returns immediately with an error. If the call is not restarted by the kernel, `select` is called to wait for the connection to complete. 
## `wait` and `waitpid` functions 
