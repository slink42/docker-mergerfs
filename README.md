# docker-rclone

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
    -e TZ=Australia/Melbourne \
    -e MERGED_DEST=/mnt/data \
    -e MFS_BRANCHES:/mnt/local1=RW:/mnt/local12=NC \
    slink42/mergerfs
```

First, you need to prepare volume bind mounts for folders to merge.

```bash
docker-compose exec <service_name> rclone_setup
```

Then, up and run your container with a proper environment variable ```RCLONE_REMOTE_PATH``` which specifies an rclone remote path you want to mount. In the initialization process of every container start, it will check 1) existance of ```rclone.conf``` and 2) validation of ```RCLONE_REMOTE_PATH``` whether it actually exists in ```rclone.conf```. If there is any problem, please check container log by

```bash
docker logs <container name or sha1, e.g. rclone>
```

### rclone mount

Here is the internal command for rclone mount.

```bash
rclone mount ${RCLONE_REMOTE_PATH} ${mergerfs_mountpoint} \
    --uid=${PUID:-911} \
    --gid=${PGID:-911} \
    --cache-dir=/cache \
    --use-mmap \
    --allow-other \
    --umask=002 \
    --rc \
    --rc-no-auth \
    --rc-addr=:5574 \
    ${RCLONE_MOUNT_USER_OPTS}
```

Please note that variables only with capital letters are configurable by environment variables. Also, be aware that rc is enabled by default.

| ENV  | Description  | Default  |
|---|---|---|
| ```PUID``` / ```PGID```  | uid and gid for running an app  | ```911``` / ```911```  |
| ```TZ```  | timezone, required for correct timestamp in log  |   |
| ```RCLONE_REMOTE_PATH```  | this should be in ```rclone.conf```  |   |
| ```RCLONE_CONFIG```  | path to ```rclone.conf```  |  ```/config/rclone.conf``` |
| ```RCLONE_LOG_LEVEL```  | log level for rclone runtime  | ```NOTICE```  |
| ```RCLONE_LOG_FILE```  | to redirect logging to file  |   |
| ```RCLONE_MOUNT_USER_OPTS```  | additioanl arguments will be appended to the basic options in the above command  |   |

## [mergerfs](https://github.com/trapexit/mergerfs) or unionfs (optional)

Along with the rclone folder, you can specify one local directory to be mergerfs with by ```POOLING_FS=mergerfs```. Internally, it will execute a following command

```bash
mergerfs \
    -o uid=${PUID:-911},gid=${PGID:-911},umask=022,allow_other \
    -o ${MFS_USER_OPTS} \
    /local=RW:/cloud=NC /data
```

where a default value of ```MFS_USER_OPTS``` is

```bash
MFS_USER_OPTS="rw,use_ino,func.getattr=newest,category.action=all,category.create=ff,cache.files=auto-full,dropcacheonclose=true"
```

If you want unionfs instead of mergerfs, set ```POOLING_FS=unionfs```, which will apply

```bash
unionfs \
    -o uid=${PUID:-911},gid=${PGID:-911},umask=022,allow_other \
    -o ${UFS_USER_OPTS} \
    /local=RW:/cloud=RO /data
```

where a default value of ```UFS_USER_OPTS``` is

```bash
UFS_USER_OPTS="cow,direct_io,nonempty,auto_cache,sync_read"
```

#### cron - disabled by default

After making sure that a single execution of scripts is okay, you can add cron jobs of these operations by setting environment variables.

| ENV  | Description  | Default  |
|---|---|---|
| ```COPY_LOCAL_SCHEDULE```  | cron schedule for copy_local  |  |
| ```MOVE_LOCAL_SCHEDULE```  | cron schedule for move_local  |  |

## Credit

- [cloud-media-scripts](https://github.com/madslundt/docker-cloud-media-scripts)
