# Assignment 2

This blogpost was written for the second assignment of the 'Big Data' course. The aim of this post is to document my experience with running my first simple Map-Reduce jobs on a pseudo distributed file system running locally in a Docker container. While working on this assignment I will also collect a list of useful commands that are not directly related to this assignment but serve as a 'cheat sheet' for working with Docker and the HDFS in general. These notes are found [here](setupInstructions.md).

## Storing data on the HDFS

In order to run a Map-Reduce job on data on the HDFS we first need data. For this purpose a file with the complete works of Shakespeare is stored on the dfs. I obtained the file from github using `curl https://raw.githubusercontent.com/rubigdata-dockerhub/hadoop-dockerfile/master/100.txt > 100.txt` (For some reason wget would only give me the source of the github page instead of the actual file). After running this command in the docker container the file is downloaded and stored in the current directory. To copy it to the running dfs I used `bin/hdfs dfs -put 100.txt input`. This command only works as intended if the folder 'input' has already been created on the dfs in the folder 'user/root' (if this folder does not exist, the command will instead create a file called 'input' and store the contents of 100.txt in this file. If this happens `hdfs dfs -rm user/root/input` can be used to delete the erroneously created file).

## Running a Map-Reduce job

Map Reduce jobs are written in Java. To run them they need to be packaged into a .jar file and can be run on the distributed file system by calling the command `hdfs dfs jar [JARFILE] [ARGUMENTS]` where the first argument is the .jar that contains the job and the remaining arguments are arguments passed to the .jar. The example given in the assignment is:

``` hadoop jar wc.jar WordCount input/100.txt output ```

The jar wc.jar contains a file named WordCount.java that contains the instructions for the actual map-reduce job. `input/100.txt` specifies the file(s) that the job is run on and `output` specifies the folder where the result of the job is stored. Both the input and output folder/file are located on the dfs, while the jar is located on my local machine.
After executing the job, `hdfs dfs -get /output` is used to copy the output folder back to the local machine for inspection.

## Writing a Map-Reduce job

The word count job used as an example in the previous section was taken from the [Map Reduce tutorial](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0). In this section I explain how to write a Map-Reduce task from scratch using the simple example of counting the total number of words in the works of Shakespeare. 


