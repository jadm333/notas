# Docker

## Alternatives for persistent data

[Mount options](types-of-mounts-bind.png)

### `tmpfs` Mounts
The directories mounted in tmpfs appear as
a mounted filesystem but are stored in memory, not to persistent storage
such as a disk drive.


### `Bind` Mounts
In bind mounts, the file/directory on the host machine is mounted into
the container.By contrast, when using a Docker volume, a new directory
is created within Docker’s storage directory on the Docker host and the
contents of the directory are managed by Docker.

### `Volumes`
Docker volumes are the current recommended method of persisting data
stored in containers. Volumes are completely managed by Docker and
have many advantages over bind mounts:

- Volumes are easier to back up or transfer than bind mounts
- Volumes work on both Linux and Windows containers
- Volumes can be shared among multiple containers without problems


## Networks

### Bridge Network

A bridge network is a user-defined network that allows for all containers
connected on the same network to communicate.
## Networks

PENDING
