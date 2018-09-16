---
layout: post
title: "Installing sshfs in windows and edit remotely with pycharm"
---

# Installing SSHFS

The process has been a real pain so I need to note it down. It could have been a lot easier I could have found this [link](https://github.com/feo-cz/win-sshfs/issues/143) earlier. Basicallly the state of sshfs is a mess, and it comes down to that sshfs windows interface can only with a specific version of dokany driver

* [Link](https://github.com/dokan-dev/dokany/releases/download/v1.0.0/DokanSetup_redist-1.0.0.5000.exe) to dokan driver 1.0.0 

* [Link](https://github.com/feo-cz/win-sshfs/releases/download/1.6.1/WinSSHFS-1.6.1.13-devel.msi) to winSSHFS 1.6.1.13 

Another thing that I needed to take care was that when I run the msi file to install WinSSHFS, the installer seems to crash midway without any indication why. I need to extract the msi package manually using this command. This will extracts the file out into a folder, and then you can run the exe file from there

```
msiexec /a C:\Users\khoa\Downloads\WinSSHFS-1.6.1.13-devel.msi /qb TARGETDIR=C:\sshfs
```

If you were like me and install packages randomly and so happened to intall Dokan version 0.7.4, the bad news is you will not be able to remove the package completely. After installing the package, there will still be a dokan.sys file in the system32 folder and you have to delete it manually. I'm unsure how to delete the Dokan entry in the software list in Settings.

# Configure remote intepreter for pycharm (professional)

Remember that remote intepreter is only supported in pycharm professional edition. You could follow this [link](https://www.jetbrains.com/help/pycharm/configuring-remote-interpreters-via-ssh.html). 

Just one small note to remember when you setup is that you should choose the *private* key file when connecting using ssh. 
