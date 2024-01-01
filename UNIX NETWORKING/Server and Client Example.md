
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




