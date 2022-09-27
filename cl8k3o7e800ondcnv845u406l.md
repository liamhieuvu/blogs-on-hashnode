## Auto Clean up Docker Images on Git CI/CD Runners

If you self-host CI/CD runners for GitHub or GitLab, you may face a "full disk" problem. One of the reasons is Docker images. The runners will build Docker images before pushing them to a hub. At some points, we may need to clean up these Docker images to claim disk space for other CI/CD processes. This blog will show you the manual way, then make this automated.

# Check disk usage

First, we will check the current disk usage:

```bash
df -h
# Filesystem  Size  Used Avail Use% Mounted on
# /dev/root    62G   60G  2.1G  97% /
# devtmpfs    7.9G     0  7.9G   0% /dev
# tmpfs       7.9G     0  7.9G   0% /dev/shm
# tmpfs       1.6G  880K  1.6G   1% /run
# ...
```

To get only usage percentage of `/dev/root`:

```bash
df --output=pcent /dev/root
# Use%
# 97%
```

We will use `grep` to get the numbers only:

```bash
df --output=pcent /dev/root | grep -o '[0-9]\+'
# 97
```

# Clean up by script

We can use `docker system prune` to clean up but built docker images will boost up the building process. So, we don't want to run this command frequently. We will run this command only when the disk usage percentage surpasses a certain number, e.g. 70%.

```bash
touch ~/clean-docker.sh     # create an empty file
chmod +x ~/clean-docker.sh  # grant execution permission
```

Put the below code into `~/clean-docker.sh`:

```bash
#!/bin/bash

# save percentage of disk usage
PERCENT=$(df --output=pcent /dev/root | grep -o '[0-9]\+')

# print disk usage including date and time
echo "[$(date +"%m-%d-%Y %T")] Current disk usage: ${PERCENT}%"

# clean all unused docker data if disk usage is greater than 70%
if [[ $PERCENT -gt 70 ]]
then
  # delete all except images with lable: delete=false
  docker system prune -af --filter label!=delete=false

  # print disk usage after cleaning up
  PERCENT_AFTER=$(df --output=pcent /dev/root | grep -o '[0-9]\+')
  echo "[$(date +"%m-%d-%Y %T")] Disk usage: ${PERCENT}% --> ${PERCENT_AFTER}%"
fi
```

We can run this script whenever we want, and it cleans up Docker only if the disk usage is greater than 70%.

```bash
~/clean-docker.sh
# [09-27-2022 09:17:39] Current disk usage: 97%
# Deleted Containers:
# ...
# Total reclaimed space: 49.71GB
# [09-27-2022 09:21:41] Disk usage: 97% --> 15%

~/clean-docker.sh
# [09-27-2022 09:37:58] Current disk usage: 15%
~/clean-docker.sh
# [09-27-2022 09:38:19] Current disk usage: 15%
```

# Setup crontab for automated cleaning

We will setup crontab to run `~/clean-docker.sh` every minutes first. This help us checking the result quickly. Open crontab config:

```bash
crontab -e
```

Add this line to the config. Then, exit the editor:

```bash
* * * * * /home/ubuntu/clean-docker.sh >> /home/ubuntu/clean-docker.log 2>&1
```

+ `* * * * *` means "every minutes".
+ Use absolute path to the script. My home `~/` is `/home/ubuntu/`.
+ `>>` means appending the command's output to a file (auto create if not exist).
+ `2>&1` means redirecting stderr to stdout, so we can read both info and error logs.

After exiting the editor, we will see:

```bash
crontab: installing new crontab
```

Wait some minutes and then check `~/clean-docker.log`

```bash
[09-27-2022 09:46:01] Current disk usage: 15%
[09-27-2022 09:47:01] Current disk usage: 15%
[09-27-2022 09:48:01] Current disk usage: 15%
[09-27-2022 09:49:01] Current disk usage: 15%
[09-27-2022 09:50:01] Current disk usage: 15%
[09-27-2022 09:51:01] Current disk usage: 15%
[09-27-2022 09:52:01] Current disk usage: 15%
```

So, all things are done. If you only need to run hourly, then replace `* * * * *` by `0 * * * *`. For example, open crontab config with `crontab -e`, then edit as below:

```bash
0 * * * * /home/ubuntu/clean-docker.sh >> /home/ubuntu/clean-docker.log 2>&1
```

Thank you for reading my blog.



