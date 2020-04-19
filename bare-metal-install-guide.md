# RASP Build Guide

## Overview
This document expalins how to build a RASP System on bare metal.

## Pre-Requisities
* A server and media to build your base server:
  * Using  Red Hat Fedora Core 30. Other distributions should work, but may require some tinkering of the shared libraries for WRF and the utilities to run.
  * A CPU that supports i7 and SSE/SSE2 extensions.
  * 12Gb memory - Larger models (high resolution like 2Km or 1Km) use a lot more memory. Aim for DDR3 2400Mhz or DDR4 2400Mhz or faster.
* Ideally 500Gb or more of disk space to allow for multiple models. Each model can create up to 500Mb of data per run and with archiving set, the files system will fill up after two-three months with no deletions. To ensure that running out of disk space due to run files does not halt your system (it is good security practice as well), split the file systems up:
    * / = 40G
    * /boot = 30G
    * /var = 40G
    * swap = 12G (match the memory you have)
    * /home = 378G (what's left from 500G)
  * First time through, it may be wise to set ```SElinux``` to *permissive* and fix any warnings as they arise in use. As *root* edit the SElinux permissions file, but do not disable SELinux if at all possible (it is there to secure your host). If you are really, really sure the host will be fine, then of course you can decide to disable *SELinux*.
   * Note the number of cores you have available. Use ```cat /proc/cpuinfo | grep Mhz | wc``` to provide the number of cores you have and keep for later on for editing ```.bashrc```.

## Base Build (Fedora Core 30)
1. Take the media and build the server using the file system split as shown earlier. 
  * During the install process, set the root password to something complex and also create a user to run rasp tools, and the usual one is *rasp*. 
  * **Do not run the system as root**, as there is no need to.
2. Once the base build is on, let it reboot.
3. One rebooted, log in to the command line using *root*.
4. At the command line, get the latest patches and then restart to ensure those updates have been applied. As *root*:
  * ```dnf -y update```
  * ```reboot```
5. If this is a first host (i.e. not running *lights out*) you will need the X Windowing system to see images and use WRF tools. So install the windowing system and in this case it is KDE, but you can choose others. This takes a while as there are around 1000 packages to download and install. Then reboot again just to ensure a clean load (some servers with exotic graphics cards may need special drivers - NVidia is one of these that can have troubles). As *root*:
  * ```dnf -y group install "KDE Plasma Workspaces"```
  * ```reboot```
6. Once that is on now install the other tools required. As *root*:
  * ```dnf -y install libpng15 ImageMagick gcc gcc-gfortran php php-cli ```
  * ```dnf -y install ncl netcdf netcdf-devel netcdf-fortran netcdf-fortran-devel netcf-libs ```
  * ```dnf -y install lm_sensors firefox sendmail httpd perl cpan mailx eclipse gparted jq zziplib-devel libxml2-devel autoconf jasper ```
7. Remove NTP and add ```ntpdate``` for time synchronisation. As *root*:
  * ```dnf -y remove ntp```
  * ```dnf -y install ntpdate```
8. As *root*, add a perl utility library in CPAN for RASP to use for background task processing (mainly for the WPS system):
  * ```su```
  * ```perl -e shell -MCPAN```
  * Inside the CPAN shell: 
    * ```install CPAN```
    * ```reload cpan```
    * ```install Proc::Background```
9. As the *rasp* user, install the WRF and RASP binaries [reference here]. This will put folders such as *GM*, *lib* and *bin*, into the base directory. Other important folders are *mgmt* and the various model directories (e.g *UK12*).
10. We need to update some shared libraries that are there in Fedora Core 25, but are renamed (and newer) in Fedora Core 30. Do not try this step before adding the main RASP binaries, as the *lib* directory will not exist. As the *rasp* user:
  * ```mv ./lib/libjasper.so.1 ./lib/libjasper.so.1.notused```
  * ```ln -s /usr/lib64/libjasper.so.4 ./lib/libjasper.so.```
  * ```mv  ./lib/libgfortran.so.3  ./lib/libgfortran.so.3.notused```
  * ```ln -s /usr/lib64/libgfortran.so.5 ./lib/libgfortran.so.3```
11. As *root*, enable the ```httpd``` daemon (to ensure starts correctly) and ensure that ```systemd``` waits for the network interfaces to be up. As *root*:
  * ```firewall-cmd --add-service=http --permanent```
  * ```systemctl enable httpd```
  * ```systemctl start httpd```
  * ```systemctl enable systemd-networkd-wait-online.service```
12. As *rasp* update ```.bashrc``` as follows. As *rasp*:
  * ``` export BASEDIR=/home/rasp``` - Or set to whatever you have as the user name in use.
  * ```export PATH=$PATH:$BASEDIR/lib:$BASEDIR/bin:$BASEDIR/GM```
  * ```export LD_LIBRARY_PATH=/usr/lib64:$BASEDIR/lib```
  * ```export NETCDF```
  * ```export NCARG_ROOT=/usr```
  * ```export NCL_COMMAND=/usr/bin/ncl```
  * ```export OMP_NUM_THREADS=22``` 
  * ```export NCPUS=22``` - set this to the same as ```OMP_NUM_THREADS```.
  * ```ulimit -s unlimited```
  * ```export KMP_STACKSIZE=1000000000```
  * ```export NCARG_RANGS=$BASEDIR/lib/rangs```
13. As *rasp*, now check that the binaries in ```$BASEDIR/bin``` have the right shared libraries:
  * ```ldd $BASEDIR/bin/*``` - note if any have *not found*, these need to be fixed by finding the appropriate shared library. The required library may exist but have a higher number on the end. For example, you need ```filename.so.1``` but it may exist now as ```filename.so.3```. Create a symbolic link to that newer shared library for the one that the ldd did not find, ususally they are found in ```/usr/lib64```. Fix any that are not found before moving on.
  * ```ldd $BASEDIR/lib/*``` - as before,  if any have *not found*, these need to be fixed as well.
14. Final restart of the host.

That should be it. The next step is to test and to do that, as *rasp* use *runGM <model name>* supplied in the *bin* directory.

## Notes ##
  * You may find that if you have SELinix enabled (recommend *permissive* to fix issues then *enforcing*), you may need to permit processes to read files in certain locations. Use ```setsebool -P httpd_enable_homedirs 1``` as *root* to allow the HTTP daemon to see the results of the runs. If you move the output files elsewhere, then you may not need this.
  * As *rasp* create a crontab entry for your runs if you plan to run regularly. The format of what you run will vary on what you wish to do. But be careful when running runs as ```cron``` because the shell variables, when run interactively, will not be available to ```cron``` and so you may see path or other errors if ```cron``` is not using full path names.
 
