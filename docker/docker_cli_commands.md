## Start a new container:
-d daemon
```bash
docker run -d --name <name_of_container> <name_image:tag>
```

## Stop container:
```bash
docker stop <name_of_container>
```
## Start a stopped container:
```bash
docker start <name_of_container>
```

## Connect to cointeiner bash:
```bash
docker exec -it <name_of_container> /bin/bash -c "export TERM=xterm; exec bash"
```

## Create a tmpfs mount:

- `--tmpfs`: Mounts a tmpfs mount without allowing you to specify any configurable options, and can only be used with standalone containers.
- `--mount`: Consists of multiple key-value pairs, separated by commas and each consisting of a `<key>=<value>` tuple. The `--mount` syntax is more verbose than `--tmpfs`:
	+ The type of the mount, which can be bind, volume, or tmpfs. This topic discusses tmpfs, so the type is always tmpfs.
	+ The destination takes as its value the path where the tmpfs mount is mounted in the container. May be specified as destination, dst, or target.
	+ The tmpfs-type and tmpfs-mode options. See tmpfs options.

Add a mount in the run command:
```bash
docker run -d -it --name <container_name> --mount type=tmpfs,destination=<destination> <image:tag>
docker run -d -it --name <container_name> --tmpfs <destination> <image:tag>
```

## Create a bind mount:

- `-v` or `--volume`: Consists of three fields, separated by colon characters (`:`). The fields must be in the correct order, and the meaning of each field is not immediately obvious.

	+ In the case of bind mounts, the first field is the path to the file or directory on the host machine.
	+ The second field is the path where the file or directory is mounted in the container.
	+ The third field is optional, and is a comma-separated list of options, such as `ro`, `consistent`, `delegated`, `cached`, `z`, and `Z`. These options are discussed below.
- `--mount`: Consists of multiple key-value pairs, separated by commas and each consisting of a `<key>=<value>` tuple. The `--` syntax is more verbose than `-v` or `--volume`, but the order of the keys is not significant, and the value of the flag is easier to understand.

	+ The `type` of the mount, which can be `bind`, `volume`, or `tmpfs`. This topic discusses bind mounts, so the type is always `bind`.
	+ The `source` of the mount. For bind mounts, this is the path to the file or directory on the Docker daemon host. May be specified as `source` or `src`.
	+ The `destination` takes as its value the path where the file or directory is mounted in the container. May be specified as `destination`, `dst`, or `target`.
	+The `readonly` option, if present, causes the bind mount to be mounted into the container as read-only.
	+ The `bind-propagation` option, if present, changes the bind propagation. May be one of `rprivate`, `private`, `rshared`, `shared`, `rslave`, `slave`.
	+ The `consistency` option, if present, may be one of `consistent`, `delegated`, or `cached`. This setting only applies to Docker for Mac, and is ignored on all other platforms.
	+ The `--mount` flag does not support `z` or `Z` options for modifying selinux labels.

Add a bind mount in the run command:
```bash
docker run -d -it --name <container_name> --mount type=bind,source=<source>,target=<target> <image:tag>
docker run -d -it --name <container_name> -v <source>:<destination> <image:tag>
```

## Volumes

### Docker Volume Subcommands

#### Create Volume
The create volume command is used to create named volumes. The
most common use case is to generate a named volume.

```bash
docker volume create --name=<name of the volume> --label=<any_extra_metadata>
```

#### Inspect
The inspect command displays detailed information about a volume.

```bash
docker volume inspect <name of the volume>
```

#### List Volumes
The list volume command shows all the volumes present on the host
```bash
docker volume ls
```

#### Prune Volumes
The prune volume command removes all unused local volumes. When you use the --force flag
option, it will not ask for confirmation when the command is run.

```bash
docker volume prune <--force>
```

#### Remove Volumes
The remove volume command removes volumes whose names are
provided as parameters.

```bash
docker volume rm <name>
```

#### Start a container with a volume:
The syntax for using a volume when starting a container is nearly the same
as using a bind host.

- `-v` or `--volume`: Consists of three fields, separated by colon characters (`:`). The fields must be in the correct order, and the meaning of each field is not immediately obvious.
	+ In the case of named volumes, the first field is the name of the volume, and is unique on a given host machine. For anonymous volumes, the first field is omitted.
	+ The second field is the path where the file or directory are mounted in the container.
	+ The third field is optional, and is a comma-separated list of options, such as `ro`.

- `--mount`: Consists of multiple key-value pairs, separated by commas and each consisting of a `<key>=<value>` tuple. The `--mount` syntax is more verbose than `-v` or `--volume`, but the order of the keys is not significant, and the value of the flag is easier to understand.
    + The `type` of the mount, which can be `bind`, `volume`, or `tmpfs`. This topic discusses volumes, so the type is always `volume`.
    + The `source` of the mount. For named volumes, this is the name of the volume. For anonymous volumes, this field is omitted. May be specified as `source` or `src`.
    + The `destination` takes as its value the path where the file or directory is mounted in the container. May be specified as `destination`, `dst`, or `target`.
    + The `readonly` option, if present, causes the bind mount to be mounted into the container as read-only.
    + The `volume-opt` option, which can be specified more than once, takes a key-value pair consisting of the option name and its value.


```bash
docker run -it --name <volumen-name> --mount target=<target path> <image>
docker run -it --name <volumen-name> -v:<target path>
```


## Networks

PENDING

## Docker compose

### build

The build command reads the Compose file, scans for build keys, and
then proceeds to build the image and tag the image. The images are tagged
as project_service.

```bash
docker-compose build <options> <service...>
```
If the service name is provided, Docker will proceed to build the image
for just that service; otherwise, it will build images for all the services.
Some of the commonly used options are as follows:
```bash
--compress: Compresses the build context
--no-cache Ignore the build cache when building the image
```

### down
The down command stops the containers and will proceed to remove the
containers, volumes, and networks
```bash
docker-compose down
```

### exec
The Compose exec command is equivalent to the Docker exec command.
It lets you run ad hoc commands on any of the containers.
```bash
docker-compose exec  <service> <command>
```
### logs
The logs command displays the log output from all the services.
The -f option
follows the log output.
```bash
docker-compose logs <options> <service>
```
### stop

```bash
docker-compose stop
```








