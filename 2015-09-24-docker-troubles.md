# docker troubles "too many links"

Today, I wanted to setup some Gitlab build slaves, and that's where the yak shaving began. For some reason, the gitlab ci runner wants to pull the `gitlab/gitlab-runner:cache` image for docker. Ok - cool. Yet, the build runner ended up hanging at that line. Checking on the machine and trying to pull that image manually resulted in the same behavior. Looking at the docker hub image, we find out that the image in question is based on the busybox image.

On all of our provisioned machines, a `docker pull busybox` (and images based on it), resulted in errors that would result in something like `ApplyLayer status 1: stdout: stderr: link /bin/[ /bin/run-parts: too many links`. In a more recent docker client, these errors can only be seen in a server started with the `--debug=True` flag, since they currently eat up a lot of error messages when you are using a v2 registry. Since I wasn't clear on whether the error had something to do with our (minor) additions to standard docker, I downgraded to stock docker (1.6.1-something, our build is based on the latest released version, usually).

    [root ~]# docker pull busybox
    latest: Pulling from busybox
    cfa753dfea5e: Extracting [==================================================>] 666.8 kB/666.8 kB
    cfa753dfea5e: Error downloading dependent layers 
    d7057cb02084: Error pulling image (latest) from busybox, endpoint: https://registry-1.docker.io/v1/, ApplyLayer exit status 1 stdout:  stderr: ld7057cb02084: Error pulling image (latest) from busybox, ApplyLayer exit status 1 stdout:  stderr: link /bin/[ /bin/run-parts: too many links 
    FATA[0039] Error pulling image (latest) from busybox, ApplyLayer exit status 1 stdout:  stderr: link /bin/[ /bin/run-parts: too many links 

Tracking down the code of ApplyLayer in the docker source [here](https://github.com/docker/docker/blob/release/1.8/pkg/chrootarchive/init_unix.go) resulted in the realization that docker was using a chroot-based mechanism to decompress its layers. At first I assumed docker would decompress its layers in the TMPDIR of the client, but that wasn't the case. Also, it looked like this issue was basically happening only for the busybox image - other, much larger images where not affected. 

To debug this issue, I had to dig more into what was happening when the layers are untarred. I briefly thought about trying to debug the docker binary, but my golang skills are not up-to-par just yet (I'm working on it). So I decided to go with one of my more generic favorites for figuring out issues: `strace $some_binary`. It allows you to trace syscalls made by the program, and gives us a rough feeling of how a binary interacts with the system. I was hoping to somehow find  more clues towards the error that was presented to me by docker. Fortunately, using `strace -f` (`-f` to follow forked child processes) led me to realize that a lot of links are generated when the failing layer is untarred. 
First, I checked the ext4 device stats (`tune2fs -l /path/to/device`) to see whether my root partition was potentially running out of inodes, which wasn't the case, but would be the only real reason for the error I was seeing (based on some other reports around the net). Re-checking the logs obtained from strace with this finding in mind, I looked for the creation of the chroot - and realized that docker was creating a chroot on a subdirectory of the btrfs-mounted `/var/lib/docker/`. 

Now I felt like I was on track for a solution to my problem. I googled for `btrfs link problems` and found a [bugreport](https://bugzilla.kernel.org/show_bug.cgi?id=15762) for the low number of links in btrfs, which has been tuned over time. Since we run a relatively old version of btrfs (0.2, which is part of ORACLE Linux 6u5), it was entirely possible that our btrfs partitions had not been created with "extended inode refs". So unmounting and running `btrfstune -r /device` ended up solving the issue. 

    [root ~]# docker pull busybox
    latest: Pulling from busybox
    cfa753dfea5e: Pull complete 
    d7057cb02084: Pull complete 
    Status: Downloaded newer image for busybox:latest

