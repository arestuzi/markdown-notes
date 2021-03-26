# Basic of Docker Command-Line Interface

1. Create a Docker container.

    ```bash
    $ docker container run -d --name trivia fundamentalsofdocker/trivia:ed2

    # -d - "tells Docker to run the process running in the container as a Linux Daemon."
    # --name - used to give the container an explicit name.
    # ed2: is a tag just tells us that this image has been created for the second edition of this book.
    ```

2. Check the status of Docker container.

    ```bash
    ➜  docker docker container ls -l
    CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS              PORTS               NAMES
    05b71a93ec7b        fundamentalsofdocker/trivia:ed2   "/bin/sh -c 'source …"   6 minutes ago       Up 6 minutes                            trivia

    # STATUS: the container has been up and running for 9 seconds now.
    ```

3. How to stop and remove the trivia container

    ```bash
    ➜  docker docker rm -f trivia
    trivia
    ```

4. How to list containers

    ```bash
    ➜  docker docker container ls
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    ➜  docker

    # Container ID: the unique ID of the container. It is an SHA-256
    # Image: This is the name of the container image from which this container is instantiated.
    # Command: This is the command that is used to run the main process in the container.
    # Created: This is the date and time when the container was created.
    # Status: the status of the container(created, restarting, running, removing, paused, existed or dead)
    # Ports: This is the list of container ports that have been mapped to the host.
    # Names: the name assigned to this container
    ```

5. How to list not only currently running containers but all containers that are defined on the system.

    ```bash
    ➜  ~ docker container ls -a
    CONTAINER ID        IMAGE               COMMAND                 CREATED             STATUS                   PORTS               NAMES
    2bcb1db3480a        centos              "ping -c 5 127.0.0.1"   5 hours ago         Exited (0) 5 hours ago                       magical_mclean
    499f9c3aab2c        alpine              "echo 'Hello World'"    5 hours ago         Exited (0) 5 hours ago                       jolly_taussig
    ```

6. How to list the IDs of all containers.

    ```bash
    ➜  ~ docker container ls -qa
    2bcb1db3480a
    499f9c3aab2c
    ```
