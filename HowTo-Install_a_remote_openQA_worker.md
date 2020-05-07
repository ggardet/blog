# Install a remote (cloud based) openQA worker

openQA workers can be on the same network as openQA webui server, or it can also be in the cloud.
Here, we will detail specific configurations to setup a remote cloud worker which has access only to the openQA API.


## Get API keys from openQA webui host

Create a new set of API keys from (https://openqa.opensuse.org/api_keys)[openQA webui] or ask someone with admin permissions to create a set for you.


## Setup worker to use API keys and cache

With a remote worker, you cannot NFS mount `/var/lib/openqa/cache` from the openQA server as only the openQA API is reachable. Instead, you must use the (http://open.qa/docs/#_asset_caching)[cache service] described here:

Update `/etc/openqa/workers.ini` with:

```
[global]
HOST = http://openqa.opensuse.org http://myotheropenqa.org
CACHEDIRECTORY = /var/lib/openqa/cache
CACHELIMIT = 50 # GB, default is 50.
CACHEWORKERS = 5 # Number of parallel cache minion workers, defaults to 5
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

Now you can restart your worker(s):
```
sudo systemctl enable --now openqa-worker@1
```

## Workaround for tests and needles

Tests and needles are not part of the cache services, but are hold in GIT repositories, so you need to setup an auto-update of those repos. The easiest way is to use the `fetchneedles` script from openQA package to fecth GIT repos and create a cron job to update it often enough (say, every minute).

Install required package and run the fetch script a first time.
```
sudo zypper in --no-recommends openQA
sudo /usr/share/openqa/script/fetchneedles
```

Now, add a cron job to fecth tests/needles every minute `/etc/cron.d/fetchneedles`:
```
 -*/1    * * * *  geekotest     env updateall=1 /usr/share/openqa/script/fetchneedles
```
