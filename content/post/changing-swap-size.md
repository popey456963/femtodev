+++
layout = "post"
title = "Changing Swap Size in Ubuntu"
date = "2020-05-08"
categories = ["Bash", "Linux"]
+++

![System Resource Usage](/img/changing_swap_size/glances.png)

When your server runs out of memory, one of two things generally happens:

- The out of memory killer comes along and kills processes.
- Your system uses `swap` as a form of slower memory.

Swap is simply putting pages of memory from your RAM into a file / partition to increase the capacity of it.

You can find the amount of swap you have free by running:

```bash
free -h
```

To change it, you need to disable your existing swap and resize it:

```bash
# change this to where you wish your swapfile to be stored
SWAPFILE=/swapfile

sudo swapoff -a
# we generate a swapfile of size bs * count, in this case 8GB.
sudo dd if=/dev/zero of=$SWAPFILE bs=1G count=8
sudo chmod 0600 $SWAPFILE
sudo mkswap $SWAPFILE
sudo swapon $SWAPFILE
```

Then check to make sure your amount of swap has changed:

```bash
free -h
```
