# Dockerfile Instruction

## FROM
The `FROM` instruction tells the Docker Engine which base image to use for
subsequent instructions. Every valid Dockerfile must start with a FROM
instruction.

```dockerfile
FROM <image> [AS <name>]

FROM <image>[:<tag>] [AS <name>]

FROM <image>[@<digest>] [AS <name>]
```

## WORKDIR
WORKDIR instruction sets the current working directory for RUN, CMD,
ENTRYPOINT, COPY, and ADD instructions.

```dockerfile
WORKDIR /path/to/directory
```

## ADD and COPY

`COPY` supports basic copying of files to the container, while `ADD` has support for
features like tarball auto extraction and remote URL support.

```dockerfile
ADD <source> <destination>
COPY <source> <destination>
```
Change the owner/group of the files being added to the container.

```dockerfile
ADD --chown=<user>:<group> <source> <destination>
COPY --chown=<user>:<group> <source> <destination>
```

Wildcards while specifying patterns

```dockerfile
ADD *.py /apps/
COPY *.py /apps/
```

Differences between `COPY` and `ADD`:

- If the `<destination>` does not exist in the image, it will be created.

- All new files/directories are created with UID and GID as 0, i.e., as the root user. To change this, use the --chown flag.

- If the files/directories contain special characters, they will need to be escaped.

- The `<destination>` can be an absolute or relative path. In case of relative paths, the relativeness will be inferred from the path set by the WORKDIR instruction.

- If the `<destination>` doesn’t end with a trailing slash, it will be considered a file and the contents of the `<source>` will be written into `<destination>`

- If the `<source>` is specified as a wildcard pattern, the `<destination>` must be a directory and must end with a trailing slash; otherwise, the build process will fail.

- The `<source>` must be within the build context—it cannot be a file/directory outside of the build context because the first step of a Docker build process involves sending the context directory to the Docker daemon.

- In case of the `ADD` instruction:
    + If the `<source>` is a URL and the `<destination>` is not a directory and doesn’t end with a trailing slash, the file is downloaded from the URL and copied into `<destination>`.
    + If the `<source>` is a URL and the `<destination>` is a directory and ends with a trailing slash, the filename is inferred from the URL and the file is downloaded and copied to `<destination>/<filename>`.
    + If the `<source>` is a local tarball of a known compression format, the tarball is unpacked as a directory. Remote tarballs, however, are not uncompressed.

## RUN
The RUN instruction will execute any commands in a new layer on top of the current image and create a new layer that is available for the next steps in the Dockerfile

```dockerfile
RUN <command> (known as the shell form)
RUN ["executable", "parameter 1", " parameter 2"] (known as the exec form)
```
## CMD and ENTRYPOINT

`CMD` and `ENTRYPOINT` instructions define which command is executed when
running a container. The syntax for both are as follows:

```dockerfile
CMD ["executable","param1","param2"] (exec form)
CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
CMD command param1 param2 (shell form)
ENTRYPOINT ["executable", "param1", "param2"] (exec form)
ENTRYPOINT command param1 param2 (shell form)
```

| 	|No ENTRYPOINT| ENTRYPOINT exec_entry p1_entry| ENTRYPOINT [“exec_entry”, “p1_entry”]|
|---|-------------|-------------------------------|--------------------------------------|
|**No CMD** |error, not allowed| /bin/sh -c exec_entry p1_entry|exec_entry p1_entry|
|**CMD [“exec_cmd”, “p1_cmd”]**| 	exec_cmd p1_cmd |	/bin/sh -c exec_entry p1_entry |exec_entry p1_entry exec_cmd p1_cmd|
|**CMD [“p1_cmd”, “p2_cmd”]**| 	p1_cmd p2_cmd |	/bin/sh -c exec_entry p1_entry |exec_entry p1_entry p1_cmd p2_cmd|
|**CMD exec_cmd p1_cmd**| 	/bin/sh -c exec_cmd p1_cmd 	|/bin/sh -c exec_entry p1_entry |exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd|

### Gotchas About Shell and Exec Form

- In shell form, the command is run in a shell with the command as a parameter. This form provides for a shell where shell variables, subcommands, commanding piping, and chaining is possible.
- In exec form, the command does not invoke a command shell. This means that normal shell processing (such as `$VARIABLE` substitution, piping, etc.) will not work.
- A program started in shell form will run as subcommand of /bin/sh -c. This means the executable will not be running as PID and will not receive UNIX signals.

## ENV
The ENV instruction sets the environment variables to the image.

```dockerfile
ENV <key> <value>
ENV <key>=<value> ...
```
Override env var when running a containter:
```bash
docker run -it -e <env_var>="new_value" image:tag
```

## VOLUME
The VOLUME instruction tells Docker to create a directory on the host and
mount it to a path specified in the instruction.

```dockerfile
VOLUME /var/logs/nginx
```
## EXPOSE

The `EXPOSE` instruction tells Docker that the container listens for the
specified network ports at runtime.

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```
> An EXPOSE instruction doesn’t publish the port. For the port to be published to the host, you need to use the -p flag when you do a docker run to publish and map the ports.

To map the outsido port to the inside, in the example the outside port is 8080 and in the inside is 80
> -d flag is to run in the background
```bash
docker run -d -p 8080:80 image:tag
```

## LABEL
The LABEL instruction adds metadata to an image as a key/value pair.

```dockerfile
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```

Docker recommends the following guidelines:
- For Keys
    + Authors of third-party tools should prefix each key with reverse DNS notation of a domain owned by them. For example, `com.sathyasays.my-image`
    + The com.docker.*, io.docker.*, and org.dockerproject.* are reserved by Docker for internal use.
    + Label keys should begin and end with lowercase letters and should contain only lowercase alphanumeric characters, as well as the period (.) and hyphen (-) characters. Consecutive hyphens or periods are not allowed.
    + The period (.) separates namespace fields
- For values
    + Label values can contain any data type that can be represented as string, including JSON, XML, YAML, and CSV.

## Guidelines and Recommendations for Writing Dockerfiles

### Containers should be ephemeral
Docker recommends that the image generated by
Dockerfile should be as ephemeral as possible. By
this, we should be able stop, destroy, and restart
the container at any point with minimal setup and
configuration to the container.

### Keep the build context minimal
We discussed build context earlier in this chapter.
It’s important to keep the build context as minimal
as possible to reduce the build times and image size.
This can be done by using the .dockerignore file
effectively

### Use multi-stage builds
Multi-stage builds help drastically reduce the size of the image without having to write complicated scripts to transfer/keep the required artifacts.

### Skip unwanted packages
Having unwanted or nice-to-have packages increases the size of the image, introduces unwanted dependent packages, and increases the surface area for attacks.

### Minimize the number of layers
While not as big of a concern as they used to be, it’s still important to reduce the number of layers in the image. As of Docker 1.10 and above, only RUN, COPY, and ADD instructions create layers. With these in mind, having minimal instruction or combining many lines of the respective instructions will reduce the number of layers, ultimately reducing the size of the image.

## Multi-Stage Builds

[docs of multi-stage](https://docs.docker.com/develop/develop-images/multistage-build/)



