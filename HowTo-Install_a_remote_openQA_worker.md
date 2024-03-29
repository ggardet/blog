# Install a remote (cloud based) openQA worker

openQA workers can be on the same network as openQA webui server, or it can also be in the cloud.
Here, we will detail specific configurations to setup a remote cloud worker which has access only to the openQA API.

## Install required software

As any other openQA worker, you need to install some packages.
You likely want to use the latest version of openQA and thus use the binaries from `devel:openQA` and `devel:openQA:Leap:15.x` projects (adjust the URL, if you do not use Leap 15.x):
```
sudo zypper ar -f 'https://download.opensuse.org/repositories/devel:/openQA/$releasever/' devel_openQA
sudo zypper ar -f 'https://download.opensuse.org/repositories/devel:/openQA:/Leap:/$releasever/$releasever/' devel_openQA_Leap
```

If you use SLE15-SP5, you need to enable the repositories used for Leap 15.5 and also PackageHub:
```
sudo zypper ar -f https://download.opensuse.org/repositories/devel:/openQA/15.5/devel:openQA.repo
sudo zypper ar -f https://download.opensuse.org/repositories/devel:/openQA:/Leap:/15.5/15.5/devel:openQA:Leap:15.5.repo
sudo SUSEConnect -p PackageHub/15.5/aarch64
```

Now, you can install the packages:
```
sudo zypper install openQA-worker os-autoinst-distri-opensuse-deps
```


## Get API keys from openQA webui host

Create a new set of API keys from (https://openqa.opensuse.org/api_keys)[openQA webui] or ask someone with admin permissions to create a set for you.


## Setup worker to use API keys and cache

With a remote worker, you cannot NFS mount `/var/lib/openqa/cache` from the openQA server as only the openQA API is reachable. Instead, you must use the (http://open.qa/docs/#_asset_caching)[cache service] described here:

Update `/etc/openqa/workers.ini` with:

```
[global]
HOST = https://openqa.opensuse.org http://myotheropenqa.org
CACHEDIRECTORY = /var/lib/openqa/cache
# Max cache size defaults to 50 GB
CACHELIMIT = 50
# Number of parallel cache minion workers, defaults to 5
CACHEWORKERS = 5
```

Update `/etc/openqa/client.conf` with the key generated from webui:

```
[openqa.opensuse.org]
key = 0123456789ABCDEF
secret = FEDCBA9876543210
```

Start and enable the Cache Service:
```
sudo systemctl enable --now openqa-worker-cacheservice
```

Enable and start the Cache Worker:
```
sudo systemctl enable --now openqa-worker-cacheservice-minion
```


## Workaround for tests and needles

Tests and needles are not part of the cache services, but are hold in GIT repositories, so you need to setup an auto-update of those repos. The easiest way is to use the `fetchneedles` script from openQA package to fecth GIT repos and create a cron job to update it often enough (say, every minute).

Install required package and run the fetch script a first time.
```
sudo zypper in --no-recommends openQA system-user-wwwrun cron
sudo /usr/share/openqa/script/fetchneedles
```

Now, add a cron job to fecth tests/needles every minute `/etc/cron.d/fetchneedles`:
```
 -*/1    * * * *  geekotest     env updateall=1 /usr/share/openqa/script/fetchneedles
```

Fix tests and needles for alp,leap-micro and microos
```
pushd /var/lib/openqa/share/tests/
sudo ln -s opensuse alp
sudo ln -s opensuse leap-micro
sudo ln -s opensuse microos
sudo chown -h geekotest:nogroup *
pushd opensuse/products
for file in alp microos sle sle-micro; do
  pushd $file
  sudo ln -s ../opensuse/needles/
  popd
done
popd
popd
```

And restart the service:
```
sudo systemctl restart cron
```

You likely want to stop and disable openQA related services started automatically:
```
sudo systemctl disable --now openqa-webui.service
sudo systemctl disable --now openqa-gru.service
```

## Setup HugePages for aarch64 qemu tests
With `yast2 bootloader` you can add boot options to setup huge pages (here 2x 1G):
```
default_hugepagesz=1G hugepagesz=1G hugepages=2
```

You also need to fix write permissions for openQA user with `/etc/systemd/system/openqa-hugepages-fix.service`:
```
[Unit]
Description=Systemd service to fix hugepages + qemu ram problems. See https://progress.opensuse.org/issues/53234 for details
After=dev-hugepages.mount
After=ovs-vswitchd.service

[Service]
Type=simple
ExecStart=/usr/bin/chmod o+w /dev/hugepages/

[Install]
WantedBy=multi-user.target
```
And enable it with:
```
sudo systemctl enable openqa-hugepages-fix.service
```
And `reboot` your machine.


## Enjoy your remote worker


Now you can (re)start your worker(s):
```
sudo systemctl enable --now openqa-worker@1
```

And, your remote worker should be registred on your openQA server. Check the `/admin/workers` page from the openQA webUI.

Enjoy!
