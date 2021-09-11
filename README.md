# scratch

`scratch` is a tool that creates a shell environment where most filesystem modifications are not persisted.

It leverages [overlayfs](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html) to redirect all filesystem changes to a temporary filesystem in `/tmp/scratch-envs`.

## Why?

Routine actions sometimes require too high of a commitment:
* Afraid a system update may screw up the package manager, and want get a feel of things first?
* Want to temporarily install something that brings a million dependencies, and want a way to completely clean it up without having to remember how?
* Want to do `rm -rf *` for therapy or for fun, but without living the consequences?
* Afraid of the impact of a `git` command, and want to try it first to see its effects?

In all of these cases, `scratch` can help. Actions perpetrated inside the scratch session will not visible outside.

## How to use `scratch`?

Simply do `sudo scratch <env name>`, where `<env name>` is a name of your choice to identify the new environment. A bash session inside that scratch environment will be started, and can be exited at any moment (with `exit`, CTRL-D, etc).

Changes made inside a scratch environment are "persisted" at `/tmp/scratch-envs/<env name>`. From outside the environment, changes are visible in `/tmp/scratch-envs/<env name>/top/`.

It is possible to exit and re-enter the environment, resuming where things were left off. But because `/tmp/scratch-envs` will likely be erased on reboot, the environments won't survive a reboot. Though you are free to change the location to a persistent filesystem (not in `/tmp`, which is usually cleared on reboot).

## Warning! Warning!

* Use at your own risk!
* It is truly disgusting code. The extent of my testing is "it works on my laptop".
* You may forget that you are inside (or outside) a scratch environment. Which means you may accidentally lose data (and time).
* This is especially problematic if you do not have a way to see if you're inside or outside a scratch environment. You may feel emboldened enough to do `rm -rf /` outside a scratch environment, making your system unusable. Similarly, you may do all sort of nice things inside a scratch environment, only to realize you have to do it all over again outside.
* If you're modifying files inside the environment, things may be slow. Touching one byte of a single file requires copying and rewriting the whole file in the overlay upper directory.
* Do not expect scratch environments to stay stable over time. Because the base filesystem can also change underneath, the overlay filesystem will eventually be unusable due to inconsistencies. Scratch environments are meant to be relatively short-lived, they're not something you can rely upon for long-term stability.
* Everything may not work. Some software is smart enough to realize the treachery (`systemd`, for example) and will refuse to run. Some will just be confused (like software installed by `snap`). Your mileage may vary.
* `scratch` may fail to create an overlay on top of some mountpoints (due to weird mount topologies or overlayfs limitations). It should say so when it does, but you need to be extra-careful. Before doing anything destructive, make sure the overlay functionality works in the location you want to make changes in.

It is strongly recommended to display scratch information in your shell. For example, you can add something like this in your `.bashrc` to modify the prompt:
```
if [[ -f /etc/scratch-environment ]]; then
    PS1='\[\033[01;32m\]\u@\[\033[01;31m\]$(cat /etc/scratch-environment 2>/dev/null)\[\033[00m\]: \[\033[01;34m\]\w\[\033[00m\]\$ '
else
    PS1='\[\033[01;32m\]\u@\[\033[01;31m\]\h\[\033[00m\]: \[\033[01;34m\]\w\[\033[00m\]\$ '
fi
```

This makes the shell prompt display the hostname when outside a scratch environment, and display the name of the scratch environment when inside. It works by reading `/etc/scratch-environment`, which is populated by `scratch` in every scratch environment. This file contains the name of the current environment (and should not exist outside scratch environments).


## Example

Let's look at some files (`test1`, `test2`), enter a scratch environment and modify/delete them, and crerate a new one.

After exiting the environment, all changes are gone.

```
vic@lapdog: ~/test$ cat test1
test1
vic@lapdog: ~/test$ cat test2
test2
vic@lapdog: ~/test$ sudo scratch noconsequences
[...]
Entering scratch environment noconsequences
-----------------------------
vic@noconsequences: ~/test$ cat test1
test1
vic@noconsequences: ~/test$ cat test2
test2
vic@noconsequences: ~/test$ rm test1
vic@noconsequences: ~/test$ echo abcd > test2
vic@noconsequences: ~/test$ touch test3
vic@noconsequences: ~/test$ ls
test2  test3
vic@noconsequences: ~/test$ cat test2
abcd
vic@noconsequences: ~/test$
exit
-----------------------------
Leaving scratch environment noconsequences
vic@lapdog: ~/test$ ls
test1  test2
vic@lapdog: ~/test$ cat test1
test1
vic@lapdog: ~/test$ cat test2
test2
vic@lapdog: ~/test$ ls -al /tmp/scratch-envs/noconsequences/top/home/vic/test/
total 4
drwxr-xr-x 2 vic  vic   100 Sep 10 19:06 .
drwxr-xr-x 4 vic  vic   140 Sep 10 19:43 ..
c--------- 1 root root 0, 0 Sep 10 19:06 test1 # The "deleted" marker for test1
-rw-r--r-- 1 vic  vic     5 Sep 10 19:06 test2
-rw-r--r-- 1 vic  vic     0 Sep 10 19:06 test3
```

## How does it work?

What `scratch` does is:
1. Inside a new child mount namespace, create an overlay filesystem for every mount point (including the root filesystem), with the upper dir in `/tmp/scratch-envs`.  
   There are some exceptions - notably, `/dev`, `/proc`, `/sys` and `/run` are bind mounts, not overlay mounts. This is to ensure the system stays usable in common cases.
2. Create and enter another child mount namespace, where the root is the root of the overlay filesystem for `/`. Think of it as a `chroot` step.

