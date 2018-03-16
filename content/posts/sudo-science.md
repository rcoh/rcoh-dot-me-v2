---
title: "Sudo Science: A brief tour of how Unix enables regular users to gain super (user) powers"
date: 2018-03-13T17:56:00-07:00
draft: false
tags: ["no-magic", "operating-systems", "internals"]
---
Somehow I made it this far without actually understanding how `sudo` works. For years, I've just typed `sudo`, typed my password, and revelled in my new, magical, root super powers. The other day and I finally looked into it -- to be honest, the mechanism is not at all what I expected. After going through the basics, we'll walk through creating our own version of `sudo`.

## How Sudo Works
`sudo` is just a regular old program that essentially does 3 things:

1. Ask for your password and compare it to `/etc/shadow`
2. Check to see if your username or group is in `/etc/sudoers` file.
3. `exec` the command you want to run as `root`

`root` is a special user that is bestowed with super powers, among them, impersonating other users.
The key is that `sudo` is a `setuid` binary -- this means that it gets run as the user that owns it _instead_ of the current user. **This is what gives `sudo` its power**. Because `sudo` is owned by `root`, the program is run as root.

```bash
✗ ls -l /usr/bin/sudo
-rwsr-xr-x 1 root root 136808 Jul  4  2017 /usr/bin/sudo
   ^
   The magic bit here is "s" making it a setuid binary
```

## Let's Make A (useless & dangerous) Sudo
`setuid` is actually a bit subtle. To explore the subtleties, we'll go through a few false starts on our path to implement `sudo`.

### As a bash script
Let's try writing a bash script that will make us root without asking for a password:
```bash
#!/bin/bash
exec $@
```
```bash
$ sudo chown root.root badsudo.sh
# Give the user that owns this binary, `s`, setuid permissions
$ sudo chmod u+s
```

Let's try doing something privileged:
```bash
$ russell@russell-linux:~/scratch$ ./badsudo.sh cat /etc/sudoers
cat: /etc/sudoers: Permission denied
```
Womp. Sadly, we are not yet root. The reason why this doesn't work is an important distinction: The binary in this situation is `/bin/bash`, invoked by the shebang -- we only marked the shell script as `setuid`. Since `/bin/bash` isn't a `setuid` binary, our shell script didn't run as root.

If you are a thrill seeker, you can mark `/bin/bash` as `setuid` and `chown` it to `root`. Please don't do that.

### In Rust
So we need a compiled language to make this work...to Rust!
```rust
extern crate exec;
use std::env;
fn main() {
    let args: Vec<String> = env::args().collect();
    let _res = exec::execvp(args[1], args[1..]);
}
```

Let's try it! If `whoami` prints `root`, we know we can run programs as root.
```bash
$ sudo chown root.root rustsudo
$ sudo chmod u+s rustsudo
$ ./rustsudo whoami
russell
```
Still doesn't work! Turns out Linux _really_ doesn't want you messing around with `setuid` binaries. They're just too dangerous and powerful. For example, `setuid` binaries won't work on external media because that could allow untrusted users to elevate their permissions by inserting a flashdrive. Mounting your home directory with `nosuid` is just another precaution to make it more difficult for untrusted programs to escalate their permissions.   Viewing the `mount` information for my home directory:
```bash
$ mount | grep "russell"
/home/.ecryptfs/russell/.Private on /home/russell type ecryptfs (rw,nosuid,nodev,...)
```
Note the critical `nosuid` set! To word around this, we need to put the binary in a directory _not_ mounted with setuid. On my computer at least, `/tmp` is mounted on `/`, the root filesystem. 

### In Rust, Take 2
Don't try this at home (or if you do, make sure to clean up after yourself)
```bash
$  cp rustsudo /tmp 
$  /tmp ls -l rustsudo 
-rwxr-xr-x 1 russell russell 5809968 Mar 16 11:16 rustsudo
$  sudo chown root.root /tmp/rustsudo 
[sudo] password for russell: 
➜  sudo chmod u+s /tmp/rustsudo 
➜  /tmp/rustsudo whoami
root
➜  /tmp/rustsudo cat /etc/sudoers | head
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
......
```
Success!

## Closing Thoughts
I was actually really surprised by the `setuid` mechanism that enables `sudo` to work. In hindsight it seems reasonable, but I had no idea there was a filesystem bit that marked a program to be run as it's owner. It's a really cool mechanism! If your program is running as root, it's a good idea to run `setuid` to a non-root user as soon as possible to minimize the exploit surface.
