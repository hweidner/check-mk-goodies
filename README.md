Checkmk goodies
===============

Some useful Linux checks for the [Checkmk](https://checkmk.com)
monitoring system.

Currently, all the checks are
[Checkmk local checks](https://checkmk.com/cms_localchecks.html).
They are used together with the Checkmk Linux agent.

Installation
------------

To use these scripts, you need to install the
[Linux Agent for Checkmk](https://checkmk.com/cms_agent_linux.html).

Put the scripts under the "local" directory of the Linux agent, which is
``/usr/lib/check_mk_agent/local`` on most systems. (When using CentOS/Red Hat and
installing the agent from the EPEL repositories, the directory is
``/usr/share/check-mk-agent/local``.) The scripts must be executable by the
Checkmk agent (e.g. ``chmod 755 <Checkname>``).

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
average pressure (in percent) in the last 60 and 300 seconds. (The PSI also
contain a value about the last 10 seconds, but the check omits them, due to the
default Checkmk check interval of 60s.)

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

(Note that the image shows an older version of this check, containing also
10s average values.)

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

This script should not be used on hosts that are part of a Kubernetes cluster.
On such hosts, the docker containers are managed and kept up-to-date by Kubernetes.
You might consider using the ``k8s`` local check explained below.

k8s (Kubernetes)
----------------

The k8s local check script is an alternative approach to Kubernetes monitoring.

The Checkmk monitoring system (starting with version 1.5.0p12) already offers
a comprehensive monitoring solution for Kubernetes clusters, implemented by a
special agent and several checkplugins. However, they do not fit the needs of
all installations. The Checkmk Kubernetes plugins map each Kubernetes pod,
service, deployment and other entities to an own "host" using the piggyback
mechanism. This has several implications:

  * The number of "hosts" generated by the Kubernetes plugins can be very huge.
    It is much work to create them manually and keep them up-to-date. One might
    want to enable the dynamic configuration daemon (DCD), but it is only present
    in the Checkmk enterprise edition. With the Checkmk raw (open source) edition,
    you need to create the hosts manually, or need 3rd party scripts to automate
    that.
  * Even in a small Kubernetes cluster, the number of piggyback hosts quickly
    exceeds 10, which is the host limit for the Checkmk demo edition.
  * In a Kubernetes cluster, some entities come and go. This is always true for
    pods, as replaced pods have an other, randomly generated name. In an agile
    or development environment, this can be also true for deployments, services,
    ingresses etc. The DCD typically runs every hour or every 15 minutes, but
    pods appear and vanish much faster. You will receive a lot of UNKNOWN messages
    from Checkmk, though.
  * The way of mapping Kubernetes entities to host names makes it hard to
    monitor more than one Kubernetes cluster on the same Checkmk server.
    Kubernetes system services from different clusters will appear under the
    same host name. You need complex translation rules to separate them.

In contrast to the buildin Kubernetes monitoring, the ``k8s`` local check offers
a rather conservative approach to Kubernetes monitoring.

The script is installed on one (or more) hosts of the Kubernetes cluster, or on
a separate control node which has access to the cluster. It requires the ``kubectl``
binary and a KUBECONFIG file with the credentials to access the cluster and read
the informations on all nodes, namespaces, pods, services, deployments, ingresses,
statefulsets, daemonsets, and replicasets. The output consists of a quite constant
set of check items:

  * one item for each namespace
  * one item for each node
  * one item each for a summary of pods, pods by node, and pods by namespace
  * one item each for a summary of all services, deployments, ingresses,
    statefulsets, daemonsets, and replicasets
  * one item for each namespace and its services, deployments, ingresses,
    statefulsets, daemonsets, and replicasets

The number and names of the check items are constant, unless you add or remove
cluster nodes, or create/delete namespaces. This does happen rather seldom.
In this cases, the host(s) running the local check have to be reinventorized,
potentially by an automated intentory check. As long as nodes and namespaces
are unchanged, no inventorization is needed.

Most of the check items are always in the state OK. An exception are the check
items for each namespace and deployment, statefulset, daemonset, and replicaset.
They compare the number of current, ready, and up-to-date entities to the
desired number of entities. The state is WARN when there are differences, and
CRITICAL when one of the numbers is zero but the desired number is greater.

For the check items of each namespace and its services, deployments, ingresses,
statefulsets, daemonsets, and replicasets, the long (multiline) output contains
a detailed list of the monitored Kubernetes entities. These checks also output
the number of configured entities as performance values. The summary check items
for nodes, namespaces and pods also output their respective counts as performance
values.

![k8s example](img/k8s.png "k8s example")

![k8s multiline example](img/k8s_multiline.png "k8s multiline example")

The check is installed by copying the ``k8s`` script to the local checks
directory, which is usually ``/usr/lib/check_mk_agent/local``, and made
executable (``chmod 755 k8s``). The configuration values KUBECTL und KUBECONFIG
might have to be adapted to the own installation; in the current version, they
refer to a MicroK8s installation from the snap repository of an Ubuntu system.

The k8s Check was developed and tested using Kubernetes (and kubectl) version
1.19.0 and 1.18.8. Details might differ with other versions, thus some adaptions
might be needed.

