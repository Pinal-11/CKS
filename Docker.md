To start the Docker Daemon `dockerd` for the debug mode `dockerd --debug`

whne docker daemon start, it listens on an internal unix socket at the path `/var/run/docker.sock`. 
This can be seen output of the logs 

As of now docker daemon only run on the local Docker host what if i want to run the different docker host in the my local machine like the external docker host thenn ??
`dockerd --debug --host=tcp://192.168.1.10:2375` 2475 is standard port for the docker