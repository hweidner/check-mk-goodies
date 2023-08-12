# autopb (Automatic Piggyback)

The autopb script automatically creates hosts in Checkmk when
[piggyback](https://docs.checkmk.com/latest/en/piggyback.html) data
is available. It supports only the Checkmk Raw edition, version 2.1
(developed and tested with 2.1.0p14).

This is very basically what the
[Checkmk Dynamic Configuration Daemon (DCD)](https://docs.checkmk.com/latest/en/dcd.html)
and its piggyback connector does, which is only available in the Enterprise edition.

In Checkmk, many buildin checks create piggyback data. Examples are the the
special agents for VMware, Proxmox VE or Kubernetes, or the agent plugin for Docker.
Piggyback data is data coming from one host (e.g. a Docker host or a Proxmox VE
host), but referring to another "host" object (the container or VM). Checkmk will
display these informations only if the there is a host object configured with
that name.

The `autopb` script scans the site's configuration for hosts that are not yet
created as host objects in WATO, but piggyback data is available for them. This
is done by parsing the `hosts.mk` files below `etc/check_mk/conf.d/wato`, and the
piggyback directories under `tmp/check_mk/piggyback`. It then creates a `hosts.mk`
file for a special WATO folder named `autopb`, which must be created first. The
script also removes those hosts if no more piggyback data is available or if
they are older than four hours (configurable in the beginning of the script).

If new hosts were added, a service discovery `cmk -I` is issued on them.
Whenever the hosts.mk file changes, the configuration changes are activated
with `cmk -R`, which is the recommended way on the Checkmk Raw Edition.

The automatically created host objects have their piggyback sources configured
as parents, so that the parent-child relationship is used in notifications and
in the topology graph. Besides that, they are rather dumb. They do not have an
IP address nor get their UP/DOWN state checked in any way. There are solely
there to display the collected piggyback data.

While having some shortcomings (see below), this solution makes it more
comfortable to use checks with piggyback data. It is especially helpful when
monitoring a Kubernetes cluster with Checkmk: every new deployment, service,
ingress or statefulset deliver piggyback data that will only show up when an
host object is created. On a busy cluster, this will be too much to do
manually.

## Installation

To install the script, follow the steps:

1. Install the `autopb` script under `local/bin` of your Checkmk site and make
   it executable.
2. Install the Perl Template Toolkit. E.g. on Debian or Ubuntu, do
   ```bash
   # apt install libtemplate-perl
   ```
3. In Checkmk, create a WATO folder (Setup -> Hosts -> Create Folder) named
   `autopb`. Do not manually create any hosts in that folder.
4. In Checkmk, create a rule of type _Periodic Service Discovery_ in the
   `autopb` folder. Specify that a service discovery should be performed
   every 15 minutes, and that the service configuration should be automatically
   updated in the mode _Refresh all services and host labels (Tabula rasa)_.
   Changes should be grouped for e.g. 5 minutes, and activated automatically.
5. In Checkmk, create a rule of type _Host check command_ in the `autopb` folder.
   Under _Host check command_, select the option _Always assume host to be up_.
6. Run the `local/bin/autopb` script manually once. If there is piggyback
   data available, this might take a while, as every new host is discovered.
7. In `etc/cron.d/autopb`, create a cron job that runs the script e.g. every 15
   minutes:
   ```
   */15 * * * *  $HOME/local/bin/autopb
   ```
   Reload the crontab with `omd reload crontab`.

## Caveats

This solution has some shortcomings. Compared to the DCD, it takes much longer
for piggybacked hosts and their monitoring data to show up in Checkmk. With the
configuration shown above, it can take up to 15 minutes for a host to show up,
and some more time until all checks are visible.

Every automatically created host object belongs to the `autopb` WATO folder.
This is different from the DCD, where it is possible to attach the dynamically
created hosts to different folders, depending on the host where the piggyback
data originated.
The Checkmk Enterprise edition and the DCD approach gives you more power to avoid
naming conflicts, e.g. by translating hostnames according to the folder. With
the autobp solution, it can be difficult to monitor several Docker hosts or
hypervisors when the names of containers or VMs do overlap.

If you get a host object automatically created by this mechanism, you might
later want to create it as a "regular" host. For example, if you run a
Proxmox VE host and create a VM, you might later want to actively check it with
Ping or query a Checkmk agent on the VM. Note that you cannot create a second
host with the same name in Checkmk. So you either need to move the host into an
other WATO folder, and make sure that the settings regarding IP address and
Agent are changed. If you decide to delete the host from the `autopb` WATO folder
and create a new host in another folder, make sure that this happens not interrupted
by a new run of the `autopb` script. Otherwise, the host object would be created
again.

## Notes for Checkmk Docker image

When you run Checkmk from the official Docker image, the Perl Template Toolkit
is not installed. You can install it manually (the image is based on Debian 10),
but this needs to be repeated on every new release.

To automate this, write a script that starts the container and immediately
installs the needed packages, like this:

```bash
docker run -d --name checkmk ... checkmk/check-mk-raw:2.2.0-latest
docker exec checkmk -c 'apt-get update && apt-get -y install libtemplate-perl'
```

If you use [watchtower](https://containrrr.dev/watchtower/) to continuously update
running Docker containers to the latest images, you can signal watchtower to install
the package after each update by setting a label, e.g.

```bash
docker run -d --name checkmk ... \
  --label=com.centurylinklabs.watchtower.lifecycle.post-update="apt-get update && apt-get -y install libtemplate-perl" \
  checkmk/check-mk-raw:2.2.0-latest
docker exec checkmk -c 'apt-get update && apt-get -y install libtemplate-perl'
```

Btw, this method can also be used to install other useful packages into the Checkmk container,
like `less` or a proper editor like `nano` or `joe`.
