# Hadoop and Map Reduce

## Setting up the pseudo cluster

### Downloading the Docker container

The pseudo cluster for this course is run in a prepared Docker container. To set up the container download it using `docker pull rubigdata/hadoop`

### Running the Container

The docker container is started by running `docker run -d rubigdata/hadoop` it is useful to store the result of this command in a variable to easily access the running container at a later point (the command will return the hash of the container). The `-d` flag detaches the container from the terminal it is started from so that it runs in the background.

To open a shell in the container that was just started use the command `docker exec -it $hash /bin/bash` with $hash the hash value of the container.

### HDFS Setup

1. Format the namenode. `hdfs namenode -format`
2. Create a user directory for the root user (/user/root)

### Starting the DFS

`sbin/start-dfs.sh` starts the distributed file system in the docker container.

### Copying Files to the Distributed File System

Operations on the Filesystem are performed with `hdfs dfs [OPTIONS]`. To copy files from the local filesystem to the dfs use the -put flag. (Look [here](https://hadoop.apache.org/docs/r2.7.5/hadoop-project-dist/hadoop-common/FileSystemShell.html#put) for a handy cheat sheet)

### Running an example job

Executing a jar file on the distributed file system is done with `hadoop jar [FILE]`. In the example given `hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.9.2.jar grep input output 'dfs[a-z.]+'` executes the jar called hadoop-mapreduce-examples-2.9.2.jar on the distributed file system. With the parameters supplied the output is saved in the folder output.

To inspect the results of this job the contents of output are copied from the distributed file system to the local filesystem (in the docker container) with `hdfs dfs -get output output`. This copies the output folder on the dfs to a folder called output in the docker container.
