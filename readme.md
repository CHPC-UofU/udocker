# Udocker

## Installation

First install in user space, following https://github.com/indigo-dc/udocker/blob/master/doc/installation_manual.md, though the simplest worked:
```
  curl https://raw.githubusercontent.com/indigo-dc/udocker/master/udocker.py > udocker
  chmod u+rx ./udocker
  ./udocker install
```

```~/.udocker``` contains bin and lib directories with dependencies, such as proot and runc, which provide the chroot capability. It also hosts the docker style local container repository.

User interfaces everything with a single file command, ```udocker```, wherever it was installed in the user space - therefore adding a path to PATH or creating personal module file would be recommended.

### Sys branch installation

For general availability, the following works to install udocker system-wide.

After installing local version, copy the ```udocker``` file, and ```~/.udocker/bin``` and ```~/.udocker/lib``` to the sys branch version.

Or, alternatively, and perhaps even better - without the need for user space installation first, first create a module file for the new version that sets UDOCKER_BIN and UDOCKER_LIB, curl the udocker file to the destination in the sys branch, and run ```udocker install``` as hpcapps to install to the specified UDOCKER_BIN and UDOCKER_LIB.

In the module file, we modify the PATH and set the two variables as
```
local PATH_VERSION = "1.1.1"
local base = pathJoin("/uufs/chpc.utah.edu/sys/installdir/udocker", PATH_VERSION)
prepend_path("PATH",base)
setenv("UDOCKER_LIB",pathJoin(base,"lib"))
setenv("UDOCKER_BIN",pathJoin(base,"bin"))
```

### udocker configuration

Done with environment variables or udocker.conf, udocker.conf modifies the Config class attributes:
```
       Config.topdir = os.getenv("UDOCKER_DIR", Config.topdir)
       Config.bindir = os.getenv("UDOCKER_BIN", Config.bindir)
       Config.libdir = os.getenv("UDOCKER_LIB", Config.libdir)
       Config.reposdir = os.getenv("UDOCKER_REPOS", Config.reposdir)
       Config.layersdir = os.getenv("UDOCKER_LAYERS", Config.layersdir)
       Config.containersdir = os.getenv("UDOCKER_CONTAINERS", Config.containersdir)
       Config.dockerio_index_url = os.getenv("UDOCKER_INDEX", Config.dockerio_index_url)
       Config.dockerio_registry_url = os.getenv("UDOCKER_REGISTRY", Config.dockerio_registry_url)
       Config.tarball = os.getenv("UDOCKER_TARBALL", Config.tarball)
       Config.fakechroot_so = os.getenv("UDOCKER_FAKECHROOT_SO", Config.fakechroot_so)
       Config.tmpdir = os.getenv("UDOCKER_TMP", Config.tmpdir)
       Config.keystore = os.getenv("UDOCKER_KEYSTORE", Config.keystore)a
       ...
       if not self.reposdir:
           self.reposdir = self.topdir + "/repos"
       if not self.layersdir:
           self.layersdir = self.topdir + "/layers"
       if not self.containersdir:
           self.containersdir = self.topdir + "/containers"
```

Our current choice is to use the environment variables. UDOCKER_BIN and UDOCKER_LIB is set by us (points to the bin and lib files of the installation in the sys branch). We leave default UDOCKER_DIR=~/.udocker, where the container pieces (repos, layers, containers) go, but, alert them to preferably set UDOCKER_DIR to another location in order to reduce chances of reaching the home directory quota.

## Implementation

udocker is written in Python and runs on CentOS7 stock Python 2. Currently it does not support Python3. A suggested workaround is to change the runner in the udocker script file from python to python2. We have done this at CHPC.

udocker pulls and expands the Docker containers into a directory structure and then chroots to them. By default, it does chroot with [proot](https://proot-me.github.io/), which does not introduce any new SetUID or other escallation. Other options are Fakeroot, runC and Singularity, but, I would think that for our purposes the default should be sufficient. The chroot method can be changed with ```--execmode```. This can be also made into a new default, e.g. to change to runC:
```
$ ./udocker setup  --execmode=R1  myfed
```

One caveat with the user based installation is the default location of the container repository ```~/.udocker```, which can fill up users home directory quickly if not kept in check. We instruct users to set UDOCKER_DIR to a place that has larger size allowance, or, at lease, where it is more visible so one is not surprised when quota is exceeded and most of the space taken is in a hidden ```~/.udocker``` directory.

## Running

Pulling containers from Docker hub is very easy:
```
$ ./udocker run ubuntu
...
 executing: bash
root@tallboy:/# 

$ ./udocker run --user=u0101881 --bindhome ubuntu
 executing: bash
u0101881@tallboy:/$ 
```
-> to bind a home directory in /uufs tree,  use ```--bindhome```
```
$ ./udocker run --user=u0101881 --bindhome --workdir=/uufs/chpc.utah.edu/common/home/u0101881 ubuntu
 executing: bash
u0101881@kingspeak1:/uufs/chpc.utah.edu/common/home/u0101881$ 
```
-> udocker by default starts with /, to start in ones home (or elsewhere), use ```--workdir```

To map /scratch, we need to add bind mount that as well:
```
./udocker run --user=u0101881 --bindhome --workdir=/uufs/chpc.utah.edu/common/home/u0101881 --volume=/scratch:/scratch ubuntu
```

To query available containers on dockerhub:
```
$./udocker search tensorflow
...
```

Users can also create containers based on Docker repos:
```
$./udocker create --name=mytf tensorflow/tensorflow
b3fdea2a-5b33-3283-aa85-6d0d777c1be0
```

and then run them (which will be faster than running from Dockerhub, even with the locally cached container image):
```
./udocker run --user=u0101881 --bindhome mytf python
...
>>> import tensorflow as tf
...
>>> sess=tf.Session()
2018-04-09 23:04:44.503864: I tensorflow/core/platform/cpu_feature_guard.cc:140] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 AVX512F FMA
```

## Modifying existing containers

The udocker documentation maintains that user can be root inside the container ```--user=root``` option and modify files as root in the container (e.g. install new packages), but, this runs into permission problems with Ubuntus apt-get (see below for how to work around that). On the other hand, this has worked with Fedora, e.g.:
```
$ ./udocker run fedora:latest
a92387cd# which vim
bash: which: command not found
a92387cd# yum install vim
...
Complete!
a92387cd# which vim
/usr/bin/vim
```

But this change is not persistent...

This can be circumvented by creating own container:
```
$ ./udocker create --name=myfed fedora:latest
$ ./udocker run --user=root myfed
a92387cd# yum install -y vim
a92387cd# exit
$ ./udocker run --user=u0101881 --bindhome myfed
adba7d51$ which vim
/usr/bin/vim
```

Note that I was able to install in RedHat based containers using yum, but, not in Debian (Ubuntu) based containers using apt, there I get an error like:
```
$ udocker run ubuntu:latest bash -c "apt update ; apt install -y cowsay"
...
E: Method gave invalid 400 URI Failure message: Could not switch saved set-user-ID
```
The relevant code in apt seems to be this:
```
 3031      return _error->Errno("getresuid", "Could not get saved set-user-ID");
 3032       if (suid != pw->pw_uid)
 3033      return _error->Error("Could not switch saved set-user-ID");
```

After some digging, the cause for this is that apt by default installs as user "_apt", not as root, and this is causing trouble with the userID mapping inside of the container. The workaround is to set the apt user to root with option ```APT::Sandbox::User=root```, e.g.:
```
apt-get -o APT::Sandbox::User=root update
apt-get -o APT::Sandbox::User=root install vim
```

Other examples, including MPI and GPU support are at that [User manual](https://indigo-dc.gitbooks.io/udocker/content/doc/user_manual.html).


# More information
[GitHub page](https://github.com/indigo-dc/udocker)
[Documentation](https://indigo-dc.gitbooks.io/udocker/content/)




