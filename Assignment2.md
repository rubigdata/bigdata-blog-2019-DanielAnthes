# Assignment 2

This blogpost was written for the second assignment of the 'Big Data' course. The aim of this post is to document my experience with running my first simple Map-Reduce jobs on a pseudo distributed file system running locally in a Docker container. While working on this assignment I will also collect a list of useful commands that are not directly related to this assignment but serve as a 'cheat sheet' for working with Docker and the HDFS in general. These notes are found [here](setupInstructions.md).

## Interacting with the Distributed File System

In order to run a Map-Reduce job on data on the HDFS we first need data. For this purpose a file with the complete works of Shakespeare is stored on the dfs. I obtained the file from github using `curl https://raw.githubusercontent.com/rubigdata-dockerhub/hadoop-dockerfile/master/100.txt > 100.txt` (For some reason wget would only give me the source of the github page instead of the actual file). After running this command in the docker container the file is downloaded and stored in the current directory. To copy it to the running dfs I used `bin/hdfs dfs -put 100.txt input`. This command only works as intended if the folder 'input' has already been created on the dfs in the folder 'user/root' (if this folder does not exist, the command will instead create a file called 'input' and store the contents of 100.txt in this file. If this happens `hdfs dfs -rm user/root/input` can be used to delete the erroneously created file).

In general, to interact with data on the distributed file system `hdfs dfs [OPTION]` is used. Depending on which argument is passed to the dfs command, different actions are performed (the commands have similar names and behave similarly to the ones used in a regular bash shell). Examples include -ls to list a directory, -rm to delete files (with the -r flag to remove folders) and in particular -get and -put to copy files between the distributed filesystem and the local machine.

## Running a Map-Reduce job

Map Reduce jobs are written in Java. To run them they need to be packaged into a .jar file and can be run on the distributed file system by calling the command `hdfs dfs jar [JARFILE] [ARGUMENTS]` where the first argument is the .jar that contains the job and the remaining arguments are arguments expected by the java program that defines the map reduce job. The example given in the assignment is:

```
hadoop jar wc.jar WordCount input/100.txt output
```

The jar wc.jar contains a file named WordCount.java that contains the instructions for the actual map-reduce job. `input/100.txt` specifies the file(s) that the job is run on and `output` specifies the folder where the result of the job is stored. Both the input and output folder/file are located on the dfs, while the jar is located on my local machine.
After executing the job, `hdfs dfs -get /output` is used to copy the output folder back to the local machine for inspection. (As an alternative it is also possible to use `hdfs dfs -cat /user/root/output/part-r-00000` to inspect the contents of the output file directly in the terminal without having to copy the file)

## Writing a Map-Reduce job

The word count job used as an example in the previous section was taken from the [Map Reduce tutorial](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0). In this section I describe the main components of a Map-Reduce task using the simple example of counting the total number of mentions of 'Othello' in the works of Shakespeare. 

### Main Components of a Map-Reduce job

A map-reduce job consists of several main components: the **mapper**,  **reducer** and optionally the **combiners**. 
When executing the map-reduce job, first the mapper instances are applied to the data. Multiple mapper instances are executed on blocks of data (possibly on multiple data nodes) in parallel. All mapper instances operate on input given in the form of key, value pairs and produce intermediate key, value pairs as their output.

In the case of the Othello Counter example (adopted from the WordCounter given as an example [here](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html#Example:_WordCount_v1.0)) the mapper splits up the input text into words and then checks for each word whether the word is "Othello". For each occurrence of the word, the mapper produces a key, value pair that holds "Othello" as its key and 1 as its value - representing one occurrence of the word. In a first attempt at creating this mapper, I simply checked for each word whether it equals the target word directly. However, since the input is split into words on whitespaces most occurrences of the target word are missed since different capitalizations and combinations such as "Othello." or "Othello," do not match.
Instead, to catch all instances of the word the refined map function converts each word to lower case and then checks whether it contains "othello" as a substring.

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

The intermediate key, value pairs are further processed by the reducers that in turn produce the final key, value pairs as their output. In this case the job of the reducer is very simple: it adds up the counts of all keys (of which in this case there is exactly one, "Othello"). The result is a set of final key, value pairs - in this case there will be exactly one pair, of which the value is the number of times "Othello" occurs in the corpus.

```
public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }
```

Since it is expensive to send many key value pairs from the mappers to the reducers, a combiner can be deployed. The combiners aggregate key, value pairs with the same key for one mapper before the pairs are passed on to the reducers. Since the map reduce framework does not guarantee the execution ofthe combiners, they need to have the same input and output types (so that successful execution of the job does not depend on the combiners being used at all). In this particular example the combiner is the same as the reducer. This is only possible because the reducer in this case has the same input and output types. This does not always need to be the case.
Why does it still make sense to use a combiner if it is identical to the reducer? - The combiners run on the same machine as the mapper and are applied before the intermediate key, value pairs are sent to the reducers that do not have to be located on the same machine. So in this case by accumulating all < "Othello", one > keypairs into a single pair the amount of data sent over the network can be greatly reduced.

### Summary

To summarize, when the map reduce job is run the following things happen:

1. The mappers process the input data and emit intermediate key, value pairs. In this case all key value pairs have the following form < Othello, 1 > 
2. (optional) The combiners combine key, value pairs with the same key for efficiency
3. The intermediate key, value pairs are shuffeled to the reducers. Critically, a reducer receives all key value pairs with the same key. This step is handled by the HDFS framework and therefore no code for this has to be written when creating a Map-Reduce job. 
4. The reducer creates the final key, value pairs that make up the output of the job.

## Romeo or Juliet?

In this section we will answer the question whether Romeo or Juliet is mentioned more often in Shakespeares work. To do so a single map reduce job is used.
The mapper will split up the text into tokens of one word each, similarly to the other jobs discussed above. Subsequently, for each word the mapper will check whether the word is either Romeo or Juliet and emit a key, value pair, where the key is either Romeo or Juliet and the value is one. In this way, only one pass over the corpus is needed to count the occurrences of both names, as opposed to for example using two separate mappers where one counts occurrences of Romeo and the other counts occurrences of Juliet in which case two passes over the data would be needed.

```
public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      Text romeo = new Text("Romeo");
      Text juliet = new Text("Juliet");

      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        if(word.toString().toLowerCase().contains("romeo"))
          context.write(romeo, one);
        else if(word.toString().toLowerCase().contains("juliet"))
          context.write(juliet, one);
      }
    }
```

The combiner and reducer remain unchanged from the Othello example. Now there are two different possible keys, so it is expected that the output will consist of two key, value pairs, where the keys are "Romeo" and "Juliet" and the values are the respective counts.

```
Juliet  206
Romeo   313
```
The above is the output generated by running this Map-Reduce job on the works of Shakespeare. Romeo is mentioned more often than Juliet, 313 times.
