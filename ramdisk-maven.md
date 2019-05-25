# Benchmark ramdisk performance for Java program compilation

## Preparation: ramdisk with tmpfs

Check whether you have SSD or not:
```
cat /sys/block/sda/queue/rotational
```
Output: 1 for hard disks and 0 for SSD.

There are different ways to make a ramdisk on Linux, here we use tmpfs.

Create a 8GB ramdisk:

```
/etc/fstab:
tmpfs /mnt/ramdisk tmpfs rw,gid=105,uid=102,size=8192M,mode=0700 0 0


```

If you decide later to increase this disk size, update fstab and run:

```
mount -o remount tmpfs
```

## Experiment 1: compare disk I/O
```
dd if=/dev/zero of=/root/testfile bs=1G count=1 oflag=direct
# 395 MB/s

dd if=/dev/zero of=/mnt/ramdisk/testfile bs=1G count=1 
# 2.1 GB/s
```


## Experiment 2: compile Java program

For simplicity of setup, use Docker container with mounted directories. 
We need to run each test twice to measure how much time is required to download components.

Setup:
- pull Maven image from Docker Hub
- clone Maven sources from GitHub to test location
- run Docker container mounting both sources and Maven cache to ramdisk

```
docker pull maven

docker run -it  -v "$(pwd)":/usr/src/mymaven -w /usr/src/mymaven -v /mnt/ramdisk/user/.m2/:/root/.m2/ maven mvn clean install

```
Note: downloaded packages are quite numerous; their total size is however just 164M.

"Download time" is difference between first two lines of the below table. 
It is time required to get all required packages one by one as they get discovered in the dependency tree and 
to write them into cache directory.

Measured time values are averages from 2 runs.

| Experiment    |    normal   |  ramdisk | 
|---------------|:-----------:|---------:|
| no .m2 cache  |     140s    |    131s  |      
| with cache    |      63s    |     60s  |
| download time |      77s    |     71s  |

## Conclusion

There is definitely a slight gain while using ramdisk, but it is also obvious that the real bottleneck is the time
required to download and cache many small packages from a remote host.

## Next steps

Repeat experiment with a local mirror. That is if you use a binary package repository software like Nexus or JFrog you
can tell it to cache packages in your local network.





