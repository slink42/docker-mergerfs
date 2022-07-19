# docker-mergerfs

[![buildx](https://github.com/slink42/docker-mergerfs/actions/workflows/buildx.yml/badge.svg?branch=master)](https://github.com/slink42/docker-mergerfs/actions/workflows/buildx.yml)

Docker image for pooling filesystem [mergerfs](https://github.com/trapexit/mergerfs/) mount or [unionfs](https://github.com/rpodgorny/unionfs-fuse/) mount, with

- Ubuntu 20.04
- some useful scripts

## Usage

```yaml
version: '3'

services:
  rclone:
    container_name: mergerfs
    image: slink42/mergerfs
    restart: always
    network_mode: "bridge"
    volumes:
      - /your/mount/points/parent/folder:/mnt:rshared
    devices:
      - /dev/fuse
    cap_add:
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=Australia/Sydney
      - MERGED_DEST=/mnt/data
      - MFS_BRANCHES:/mnt/local=RW:/mnt/cloud=NC
```

equivalently,

```bash
docker run -d \
    --name=mergerfs \
    --cap-add SYS_ADMIN \
    --device /dev/fuse \
    --security-opt apparmor=unconfined \
    -v /your/mount/points/parent/folder:/mnt:shared \
    -e PUID=${PUID} \
    -e PGID=${PGID} \
    -e TZ=Australia/Sydney \
    -e MERGED_DEST=/mnt/data \
    -e MFS_BRANCHES:/mnt/local1=NC:/mnt/local12=NC:/mnt/local13=NC \
    slink42/mergerfs
```

First, you need to prepare folders to merge within volume bind mount(s).

```bash
mkdir -p /your/mount/points/parent/folder/local1
echo 'test file contents' > /your/mount/points/parent/folder/local1/test1
mkdir -p /your/mount/points/parent/folder/local2
echo 'test file contents' > /your/mount/points/parent/folder/local2/test2
mkdir -p /your/mount/points/parent/folder/local3
echo 'test file contents' > /your/mount/points/parent/folder/local3/test3
```

Then, setup and run your container with a proper environment variable ```MFS_BRANCHES``` which specifies an branches to merge using mergerfs. Then check the logs to see if you got it right

```bash
docker logs <container name or sha1, e.g. mergerfs>
```

### mergerfs mount

Here is the internal command for mergerfs mount.

```bash
mergerfs \
    -o uid=${PUID:-911},gid=${PGID:-911},umask=022,allow_other \
    -o ${MFS_USER_OPTS} \
    /mnt/local=RW:/mnt/cloud=NC ${MERGED_DEST}
```

where a default value of ```MFS_USER_OPTS``` is

```bash
MFS_USER_OPTS="rw,use_ino,func.getattr=newest,category.action=all,category.create=ff,cache.files=auto-full,dropcacheonclose=true"
```

and the default value of ```MERGED_DEST``` is ```/mnt/data```

Please note that variables only with capital letters are configurable by environment variables. Also, be aware that rc is enabled by default.

| ENV  | Description  | Default  |
|---|---|---|
| ```PUID``` / ```PGID```  | uid and gid for running an app  | ```911``` / ```911```  |
| ```TZ```  | timezone, required for correct timestamp in log  |   |
| ```MERGED_DEST```  | local container path to merged output folder ```/mnt/data```  |   |
| ```POOLING_FS```  | pooing fs to use, mergerfs or unionfs |  ```mergerfs``` |
| ```MFS_BRANCHES```  | source banches config to pass to mergerfs | ```/mnt/local=RW:/mnt/cloud=NC```  |
| ```UFS_BRANCHES```  | source banches config to pass to unionfs | ```/mnt/local=RW:/mnt/cloud=RO```  |

If you want unionfs instead of mergerfs, set ```POOLING_FS=unionfs```, which will apply

```bash
unionfs \
    -o uid=${PUID:-911},gid=${PGID:-911},umask=022,allow_other \
    -o ${UFS_USER_OPTS} \
    /mnt/local=RW:/mnt/cloud=RO ${MERGED_DEST}
```

where the default value of ```UFS_USER_OPTS``` is

```bash
UFS_USER_OPTS="cow,direct_io,nonempty,auto_cache,sync_read"
```

and the default value of ```MERGED_DEST``` is ```/mnt/data```


## Credit

- [docker-rclone](https://github.com/wiserain/docker-rclone)
- [cloud-media-scripts](https://github.com/madslundt/docker-cloud-media-scripts)
