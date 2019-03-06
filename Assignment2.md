# Assignment 2

This blogpost was written for the second assignment of the 'Big Data' course. The aim of this post is to document my experience with running my first simple Map-Reduce jobs on a pseudo distributed file system running locally in a Docker container. While working on this assignment I will also collect a list of useful commands that are not directly related to this assignment but serve as a 'cheat sheet' for working with Docker and the HDFS in general. These notes are found [here](setupInstructions.md).

## Interacting with the Distributed File System

In order to run a Map-Reduce job on data on the HDFS we first need data. For this purpose a file with the complete works of Shakespeare is stored on the dfs. I obtained the file from github using `curl https://raw.githubusercontent.com/rubigdata-dockerhub/hadoop-dockerfile/master/100.txt > 100.txt` (For some reason wget would only give me the source of the github page instead of the actual file). After running this command in the docker container the file is downloaded and stored in the current directory. To copy it to the running dfs I used `bin/hdfs dfs -put 100.txt input`. This command only works as intended if the folder 'input' has already been created on the dfs in the folder 'user/root' (if this folder does not exist, the command will instead create a file called 'input' and store the contents of 100.txt in this file. If this happens `hdfs dfs -rm user/root/input` can be used to delete the erroneously created file).

In general, to interact with data on the distributed file system `hdfs dfs [OPTION]` is used. Depending on which argument is passed to the dfs command, different actions are performed (the commands have similar names and behave similarly to the ones used in a regular bash shell). Examples include -ls to list a directory, rm to delete files and in particular -get and -put to copy files between the distributed filesystem and the local machine.

## Running a Map-Reduce job

Map Reduce jobs are written in Java. To run them they need to be packaged into a .jar file and can be run on the distributed file system by calling the command `hdfs dfs jar [JARFILE] [ARGUMENTS]` where the first argument is the .jar that contains the job and the remaining arguments are arguments passed to the .jar. The example given in the assignment is:

```
hadoop jar wc.jar WordCount input/100.txt output
```

The jar wc.jar contains a file named WordCount.java that contains the instructions for the actual map-reduce job. `input/100.txt` specifies the file(s) that the job is run on and `output` specifies the folder where the result of the job is stored. Both the input and output folder/file are located on the dfs, while the jar is located on my local machine.
After executing the job, `hdfs dfs -get /output` is used to copy the output folder back to the local machine for inspection.

## Writing a Map-Reduce job

The word count job used as an example in the previous section was taken from the [Map Reduce tutorial](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0). In this section I describe the main components of a Map-Reduce task using the simple example of counting the total number of mentions of 'Othello' in the works of Shakespeare. 

### Main Components of a Map-Reduce job

A map-reduce job consists of several main components: the **mapper**,  **reducer** and optionally the **combiners**. 
When executing the map-reduce job, first the mapper instances are applied to the data. Multiple mapper instances are executed on blocks of data (possibly on multiple data nodes) in parallel. All mapper instances operate on input given in the form of key, value pairs and produce intermediate key, value pairs as their output.

In the case of the Othello Counter example (adopted from the WordCounter given as an example [here](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0)) the mapper splits up the input text into words and then checks for each word, whether the word is "Othello". For each occurrence of the word, the mapper produces a key, value pair that holds "Othello" as its key and 1 as its value - representing one occurrence of "Othello". In a first attemt at creating this mapper, I simply checked whether each word equals "Othello" directly. However, since the input is split into words most occurrences of the target word are missed since different capitalizations and combinations such as "Othello." or "Othello," do not match. 

```
public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      Text othello = new Text("Othello");
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        if(word.toString().toLowerCase().contains("othello"))
          context.write(othello, one);
      }
    }
```


