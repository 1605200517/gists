# Benchmarking runtime performance for Java program compilation and test

## Preparation: ramdisk with tmpfs

Check whether you have an SSD:
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

## Assessing possible optimization targets

While desired result is a production environment capable software artifact, the process can have different depth of desired reproducibility.
* Full end-to-end reproducibility requires downloading all dependencies from the original repository
* In terms of CI/CD a local or intranet cache might be enough but tests are important
* During development depending on the change footprint the developer might run only parts of the compilation and test process, for example for a single module under development. Here responsibility for complete assembly and test coverage are pushed to CI/CD which might reveal possible conflicts with other changes. 

The end-to-end process looks like the following:
* discover download dependencies if not in cache
* compile source to binary artifacts
* execute tests
* assemble bytecode to packages, depending on requirements with ot without dependencies

An optimization strategy requires facts on system's bottlenecks and measure duration of every process step.


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


| Experiment           |    normal   |  ramdisk | 
|----------------------|:-----------:|---------:|
| no .m2 cache         |     140s    |    131s  |      
| with cache           |      63s    |     60s  |
| download time        |      77s    |     71s  |
| no tests with cache  |      35s    |     35s  |
| test time with cache |      28s    |     25s  |

## Conclusion

There is definitely a slight gain while using ramdisk, but it is also obvious that the real bottleneck is the time
required to download and cache many small packages from a remote host. Also we see that the pure compilation time without tests is not depending on disk I/O. Indepedent of the storage, tests require about half of the time.
With much larger artifacts, faster storage would of course outperform the slower one, but also test and download time would increase.

The build process phases take time as the following:

| Phase                 | SSD       |   RAM   |
|-----------------------|:----------|--------:|
| Download dependencies |    55%    |   54%   | 
| Compile and package   |    25%    |   27%   |
| Test                  |    20%    |   19%   |


## Next steps

While ramdisk does not bring enough gain, there could be other optimization vectors.

For example: 

* Try parallel thread execution
* Try parallel downloads
* Repeat the experiment with a local mirror. That is if you use a binary package repository software like Nexus or JFrog you
can tell it to cache packages in your local network.





