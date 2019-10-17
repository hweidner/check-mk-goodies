Checkmk goodies
===============

Some useful Linux checks for the [Checkmk](https://checkmk.com)
monitoring system.

Currently, all the checks are
[CheckmkK local checks](https://checkmk.com/cms_localchecks.html).
They are used together with the Checkmk Linux agent.

Installation
------------

To use these scripts, you need to install the
[Linux Agent for Checkmk](https://checkmk.com/cms_agent_linux.html).

Put the scripts under the "local" directory of the Linux agent, which is
``/usr/lib/check_mk_agent/local`` on most systems. (When using CentOS/Red Hat and
installing the agent from the EPEL repositories, the directory is
``/usr/share/check-mk-agent/local``.)

osinfo
------

This shell script makes some informations about the Linux OS available to
the Checkmk monitoring console. Those are:

  * [uname](https://linux.die.net/man/1/uname) system and kernel information
  * [lsb_release](https://linux.die.net/man/1/lsb_release) information about the
    installed Linux distribution
  * [hostname](http://man7.org/linux/man-pages/man1/hostname.1.html) output
    about the system name and IP addresses
  * The kernel command line arguments use for booting (from /proc/cmdline)
  * The hypervisor, if the system is running under a hypervisor or
    container technology, and the
    [virt-what](http://people.redhat.com/rjones/virt-what/) tool is installed

The check status is always OK, entries in the monitoring console will never
get yellow or red.

The output of the osinfo script is rather static. It is therefore sufficient
to check it once an hour or even less. This can be achieved by putting the
script in a subdirecory called '3600', where the name denotes the number
of seconds until a cached output is renewed.

Example output in Checkmk console:

![osinfo example](img/osinfo.png "osinfo example")

lscpu
-----

This Perl script parses the output of the [lscpu](https://linux.die.net/man/1/lscpu)
command and makes it available on the Checkmk console. It displays the
CPU model, number of topology of virtual processors, minimum and maximum
CPU frequencies in MHz, and the size of the CPU caches.

The long (multiline) output of the check contains the whole output of the
lscpu command.

The output of the lscpu script is also rather static, it is usually sufficient
to cache it for one hour or more.

Example output in Checkmk console:

![lscpu example](img/lscpu.png "lscpu example")

![lscpu multiline example](img/lscpu_multiline.png "lscpu multiline example")

mhz
---

The ''mhz'' handles the "CPU MHz" output of the [lscpu](https://linux.die.net/man/1/lscpu)
command, which is usually the current frequency of the first processor. On modern
processors, the frequency is dynamically adjusted according to the system load.
The output is therefore dynamic, the script should be executed on every Checkmk
agent run.

The output also contains performance data, so Checkmk shows up the history
of the mhz value as a diagram.

Note that the script is usually executed in a series of other scripts/programs
started by the Linux agent. The measured CPU frequency can therefore be higher
than the average on the system.

Example output in Checkmk console:

![mhz example](img/mhz.png "mhz example")

psi
---

This local check script handles the new pressure stall information (PSI)
in the Linux kernel >= 4.20. It generated three check items for CPU, memory
and I/O pressure each. Each item output three performance values about the
average pressure (in percent) in the last 10, 60, and 300 seconds.

The memory and I/O checks give "some" and "full" values, where "some" means
that at least one process is stalled through memory or I/O pressure, while
"full" means that all processes are stalled. The CPU check has only "some"
values, as it is impossible that all processes are stalled due to CPU pressure.

See [Tracking pressure-stall information](https://lwn.net/Articles/759781/) and
[psi: pressure stall information for CPU, memory, and IO v2](https://lwn.net/Articles/759658/)
for details on how pressure stall infomation work.

As the output of this check is very dynamic, it should not be cached by the
agent (e.g. put it in the local script directory, not one of it's
subdirectories.)

Example output in Checkmk console:

![psi example](img/psi.png "psi example")

dockerpull
----------

This script checks if the currently running Docker containers have updated
images. To run the script, the images have to be pulled periodically. This
can be done by a cron job like:

	0 0 * * *  root  docker ps -qa | xargs docker inspect --format '{{.Config.Image}}' | sort -u | xargs -n1 docker pull

The local check compares the ID (SHA hash sum) of the pulled images to the
image ID of the currently running containers. If a mismatch is found, the
container name is reported and the status is red. If all containers run
on current images, the status is green.

Example output in Checkmk console:

![dockerpull example](img/dockerpull.png "dockerpull example")

