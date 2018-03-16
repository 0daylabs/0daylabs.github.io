---
author: d3adman
layout: post
title: ' Slaying the Dragon - CSAW 2015 REV 500 writeup'
categories:
- ctf-reversing
summary:  Writeup for 500 point reversing challenge wyvern
---

For this challenge we are given a binary named `wyvern`, which is a 64-bit ELF file.


	temp@temp ~/D/M/C/C/wyvern> file wyvern
	
	wyvern: ELF 64-bit LSB  executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=45f9b5b50d013fe43405dc5c7fe651c91a7a7ee8, not stripped

We are also given the following hints

* There is a dragon afoot, we need a hero. Give us the secret of the dragon and we will give you a flag.
* HINT: static is only 1 of 2 methods to RE. IDA torrent unnecessary

Before attempting to do any reversing, let us run the file first.

	temp@temp ~/D/M/C/C/wyvern> ./wyvern 
	+-----------------------+
	|    Welcome Hero       |
	+-----------------------+
	
	! Quest: there is a dragon prowling the domain.
		brute strength and magic is our only hope. Test your skill.
	
	Enter the dragons secret: a 
	
	- You have failed. The dragons power, speed and intelligence was greater.

This sample run provides us with 2 facts about the binary.

* It accepts user input
* It validates the input

We can also assume the following fact

* The length of the input is validated 

After this,we wont be able to progress much further without disassembling it.

Upon disassembling the `main()` function, we stumble across the following lines of code

{% highlight c lineos %}

.text:000000000040E1DF 1C8 mov     esi, 101h       ; n
.text:000000000040E1E4 1C8 mov     rdi, rcx        ; s
.text:000000000040E1E7 1C8 mov     [rbp+var_180], rax
.text:000000000040E1EE 1C8 mov     [rbp+var_188], rcx
.text:000000000040E1F5 1C8 call    _fgets          ; Call Procedure		//User input with limit 0x101(257)
.text:000000000040E1F5
.text:000000000040E1FA 1C8 lea     rcx, [rbp+var_120] ; Load Effective Address
.text:000000000040E201 1C8 mov     rdi, rcx
.text:000000000040E204 1C8 mov     [rbp+var_190], rax
.text:000000000040E20B 1C8 mov     [rbp+var_198], rcx
.text:000000000040E212 1C8 call    __ZNSaIcEC1Ev   ; std::allocator<char>::allocator(void)	//Creating string object using user input
.text:000000000040E217 1C8 lea     rdi, [rbp+var_118] ; Load Effective Address


.text:000000000040E242 1C8 lea     rdi, [rbp+var_138] ; this
.text:000000000040E249 1C8 lea     rsi, [rbp+var_118] ; std::string *
.text:000000000040E250 1C8 call    __ZNSsC1ERKSs   ; std::string::string(std::string const&)	//Initialize another string object with user string object 


.text:000000000040E25A 1C8 lea     rdi, [rbp+var_138]
.text:000000000040E261 1C8 call    _Z11start_questSs ; start_quest(std::string)		//Calling function start_quest with the string object containing user input
.text:000000000040E266 1C8 mov     [rbp+var_19C], eax		//Return value is stored in [rbp+var_19C]


.text:000000000040E271 1C8 mov     eax, [rbp+var_19C]
.text:000000000040E277 1C8 sub     eax, 1337h      ; Integer Subtraction	
.text:000000000040E27C 1C8 setz    cl              ; Set Byte if Zero (ZF=1)	//If eax==0x1337, cl is set
.text:000000000040E27F 1C8 lea     rdi, [rbp+var_138] ; this
.text:000000000040E286 1C8 mov     [rbp+var_1A0], eax
.text:000000000040E28C 1C8 mov     [rbp+var_1A1], cl		//cl is stored in [rbp+var_1A1]


.text:000000000040E29C 1C8 mov     al, [rbp+var_1A1]
.text:000000000040E2A2 1C8 test    al, 1           ; Logical Compare
.text:000000000040E2A4 1C8 jnz     loc_40E2AF      ; Jump if Not Zero (ZF=0)	//If cl is not set, jump to loc_40E2AF

.text:000000000040E2AA 1C8 jmp     loc_40E36C      ; Jump

.text:000000000040E36C 1C8 mov     eax, offset _ZSt4cout@@GLIBCXX_3_4
.text:000000000040E371 1C8 mov     edi, eax
.text:000000000040E373 1C8 mov     eax, offset aYouHaveFailed_ ; "\n[-] You have failed. The dragon's pow"...
.text:000000000040E378 1C8 mov     esi, eax
.text:000000000040E37A 1C8 call    __ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc ; std::operator<<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &,char const*)		//Output failed.

.text:000000000040E2AF 1C8 lea     rdi, [rbp+var_140]
.text:000000000040E2B6 1C8 lea     rsi, [rbp+var_118] ; std::string *	//Initialise another object with user input
.text:000000000040E2BD 1C8 call    __ZNSsC1ERKSs   ; std::string::string(std::string const&)

.text:000000000040E2C7 1C8 lea     rdi, [rbp+var_140]
.text:000000000040E2CE 1C8 call    _Z15reward_strengthSs ; reward_strength(std::string)	//User input string object is passed to_reward strength
.text:000000000040E2CE									//reward_strength prints flag is flag{\%s},userinput

{%endhighlight%}


From this piece of disassemble, we get the following interesting facts

* User input length should be < 127
* If return value of `start_quest(string(user input))` is `0x1337`, then the program outputs that string as flag
* Any other return value makes the program print -> You have failed

Now instead of fully disassembling start_quest, we will just note down the points that I found interesting

* 28 variables, all starting with `secret_` are pushed onto a newly created vector.
* String length of the inputted string is compared to 28
* If string length is not equal to 28, the function just returns 28 itself.

After getting all the above instructions, I decided to do run-time analysis on the binary.

I loaded up the binary in gdb and made a breakpoint,so as to read the return value of `start_quest`.
For the first run,I inputted a string of 28 `a`.

	gdb -q wyvern
	Reading symbols from wyvern...(no debugging symbols found)...done.
	(gdb) break *0x040e266
	Breakpoint 1 at 0x40e266
	(gdb) r
	Starting program: /home/lostsoul/Documents/Misc/CTF/CSAW_2015/wyvern/wyvern 
	+-----------------------+
	|    Welcome Hero       |
	+-----------------------+
	
	[!] Quest: there is a dragon prowling the domain.
		brute strength and magic is our only hope. Test your skill.
	
	Enter the dragons secret: aaaaaaaaaaaaaaaaaaaaaaaaaaaa
	
	Breakpoint 1, 0x000000000040e266 in main ()
	(gdb) p $eax
	$1 = 0
	(gdb) c
	Continuing.
	
	[-] You have failed. The dragons power, speed and intelligence was greater.
	[Inferior 1 (process 13721) exited normally]

The return value was found to be zero.After trying multiple strings, all of length 28, in all cases, the return value was zero.
Remember the vector that was created early? I tried placing the first value as the char representation of first value pushed, ie `d`.

	(gdb) r
	Starting program: /home/lostsoul/Documents/Misc/CTF/CSAW_2015/wyvern/wyvern 
	+-----------------------+
	|    Welcome Hero       |
	+-----------------------+
	
	[!] Quest: there is a dragon prowling the domain.
		brute strength and magic is our only hope. Test your skill.
	
	Enter the dragons secret: daaaaaaaaaaaaaaaaaaaaaaaaaaa
	
	Breakpoint 1, 0x000000000040e266 in main ()
	(gdb) p $eax
	$2 = 256
	(gdb) c
	Continuing.
	
	[-] You have failed. The dragons power, speed and intelligence was greater.
	[Inferior 1 (process 14525) exited normally]


Now as you see above,the value have changed.The second value pused into the vector is not a valid ascii.
So I tried to bruteforce the second character, which changed the return value and landed up in `r`.

Then I took a closer look at the values pushed, and I found out that the 2nd value pushed is a sum of ascii(d)+ascii(r).
Therefore I concluded that the values pushed into the vector are cumulative sums of ascii values of the characters of the flag.

To verify my theory, I took the 3rd value pushed on to the stack and subtracted the sum of ascii values of `d` and `r`.
Then I took the character representing the resultant value.

	(266 -(ascii(d)+ascii(r)))=52
	52 -> '4'

When I substituted the 3rd value as 4, the return value again changed, meaning it was the correct value.

After uncovering this connection, finding out the flag was trivial.

The C program I wrote for it is given below.

{% highlight c lineos %}
#include<stdio.h>

static void calc_flag(int* arr,char* result){
	int accumulator=0,i=0;
	for(i=0;i<28;i++){
		result[i]=arr[i]-accumulator;
		accumulator = accumulator+result[i];
	}
	return;
}

int main(){
	int arr[28]={
			100,214,266,369,417,527,622,733,847,
			942,1054,1106,1222,1336,1441,1540,
			1589,1686,1796,1891,1996,2112,2165,
			2260,2336,2412,2498,2575
		},i=0;
	char result[28];
	calc_flag(arr,result);
	printf("Flag is :: ");
	for(i=0;i<28;i++){
		printf("%c",result[i]);
	}
	printf("\n");
	return 0;
}

{%endhighlight%}

Compiling and running the above C code will print the flag :
	
`Flag is :: dr4g0n_or_p4tric1an_it5_LLVM`

To validate it, run the program and enter the above string.

	temp@temp ~/D/M/C/C/wyvern> ./wyvern 
	+-----------------------+
	|    Welcome Hero       |
	+-----------------------+

	[!] Quest: there is a dragon prowling the domain.
		brute strength and magic is our only hope. Test your skill.

	Enter the dragon's secret: dr4g0n_or_p4tric1an_it5_LLVM
	success

	[+] A great success! Here is a flag{dr4g0n_or_p4tric1an_it5_LLVM}

So our flag is correct !