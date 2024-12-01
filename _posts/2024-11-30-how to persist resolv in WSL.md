# How to persist /etc/resolv.conf in WSL

Recently, I found that whenever I restart WSL, the changes I made in /etc/resolve. conf are lost, and I need to configure it again manually. Finally, I found a way to persist /etc/resolv.conf in WSL. Big thanks to this [post](https://github.com/microsoft/WSL/issues/5420).

Firstly, Remove the /etc/resolv.conf:     

            sudo rm /etc/resolv.conf

And then, create a new /etc/resolv.conf:

          sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'

And then, add an entry into /etc/wsl.conf

      sudo bash -c 'echo "generateResolvConf = false" >> /etc/wsl.conf'

Finally, 

         sudo chattr +i /etc/resolv.conf

From now on, your /etc/resolv.conf will be persisted.

Enjoy it.


> [!NOTE]
> 
> chattr command is used to change the file or directory attributes. +i means "make the file immutable, it cannot be modified,
> delete, renamed, or linked"
