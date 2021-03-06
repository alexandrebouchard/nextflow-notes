# Notes on nextflow


## Setup instructions

1. Clone this repo
2. Install nextflow and make sure it's in your PATH variable
3. Create a file for your experiment, e.g. ``myExperiment.nf`` in the root
4. Use the recipe below to create your experiment
5. In the terminal, cd to the root of the repo, and type ``./myExperiment.nf -resume``




## Recipe book

### Skeleton

```
#!/usr/bin/env nextflow

deliverableDir = 'deliverables/' + workflow.scriptName.replace('.nf','')

// stuff to do

process summarizePipeline {

  cache false
  
  output:
      file 'pipeline-info.txt'
      
  publishDir deliverableDir, mode: 'copy', overwrite: true
  
  """
  echo 'scriptName: $workflow.scriptName' >> pipeline-info.txt
  echo 'start: $workflow.start' >> pipeline-info.txt
  echo 'runName: $workflow.runName' >> pipeline-info.txt
  echo 'nextflow.version: $workflow.nextflow.version' >> pipeline-info.txt
  """

}
```

### Build code

Assumes ``./gradlew installDist`` can be ran from the repo to build it (to do it, ``run gradle wrapper --gradle-version 2.14`` and commit the resulting files).

```
process buildCode {

  cache true
  
  input:
    val gitRepoName from 'rejectfree'
    val gitUser from 'alexandrebouchard'
    val codeRevision from 'XXXXX'
    val snapshotPath from '/Users/bouchard/w/rejectfree'
  
  output:
    file 'code' into code

  script:
    template 'buildRepo.sh' 
}
```

Can also use ``buildSnapshot.sh`` instead of ``buildRepo.sh`` to do quick testing before commit. Make sure to set ``cache false`` if you use the snapshot mode.



### Run the code

```
process run {

  echo true

  input:
    file code
        
  output:
    file '.' into execFolder
    
  """
  java -cp code/lib/\\* -Xmx2g XXXXX.Main  \
     --experimentConfigs.saveStandardStreams false \
     --experimentConfigs.managedExecutionFolder false 
  """
}
```


### Ranges: 

- ``each seed from 1..5``
- following does now work directly ``(0..10).collect{Math.pow(2.0, it)}`` but workaround is:
    
```
vars = (0..10).collect{Math.pow(2.0, it)}
    
process myProcess {
  input:
    each myvar from vars
    ...
}
```

### Template for aggregating

```
process analysisCode {
  input:
    val gitRepoName from 'nedry'
    val gitUser from 'alexandrebouchard'
    val codeRevision from '4c6ddf0de0027ad88d73ef6634d1e70cc9f94bfe'
    val snapshotPath from "${System.getProperty('user.home')}/w/nedry"
  output:
    file 'code' into analysisCode
  script:
    template 'buildRepo.sh'
}

process aggregate {
  input:
    file analysisCode
    file 'exec_*' from execFolder.toList()
  output:
    file 'results/latest/aggregated' into aggregated
  """
  code/bin/aggregate \
    --dataPathInEachExecFolder OUTPUT.csv \
    --keys CMD_ARG from arguments.tsv
  """
}
```
    
### Template for plotting

```
process createPlot {

  echo true

  input:
    file aggregated
    env SPARK_HOME from "${System.getProperty('user.home')}/bin/spark-2.1.0-bin-hadoop2.7"
    
   output:
    file 'times.pdf'

  publishDir deliverableDir, mode: 'copy', overwrite: true
  
  afterScript 'rm -r metastore_db; rm derby.log'
    
  """
  #!/usr/bin/env Rscript
  require("ggplot2")
  library(SparkR, lib.loc = c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib")))
  sparkR.session(master = "local[*]", sparkConfig = list(spark.driver.memory = "4g"))
  
  data <- read.df("$aggregated", "csv", header="true", inferSchema="true")
  data <- collect(data)

  p <- ggplot(data, aes(x = nUpdatedVariables, y = wallClockTimeMillis, colour = model.precision.size)) +
    geom_point() +
    scale_x_log10() +
    facet_grid(. ~ model.useLocal) +
    scale_y_log10()  
  ggsave("times.pdf", p, width = 10, height = 5, limitsize = FALSE)
  """
}
```

### Several plots for different output files on same run

```
process aggregate {

  input:
    file analysisCode
    file 'exec_*' from execFolders.toList()
    each resultFile from 'ess', 'summaryStatistics'
    
  output:
    set val(resultFile), file('aggregated') into aggregated
  
  """
  ./code/bin/csv-aggregate \
    --experimentConfigs.managedExecutionFolder false \
    --experimentConfigs.saveStandardStreams false \
    --experimentConfigs.recordExecutionInfo false \
    --argumentFileName arguments.tsv \
    --argumentsKeys XXXXX  \
    --dataPathInEachExecFolder ${resultFile}.csv
  """
}

process createPlot {

  input:
    set val(resultFile), file('aggregated') from aggregated
    env SPARK_HOME from "${System.getProperty('user.home')}/bin/spark-2.1.0-bin-hadoop2.7"
    
   output:
    file "${resultFile}.pdf"  

  publishDir deliverableDir, mode: 'copy', overwrite: true
  
  afterScript 'rm -r metastore_db; rm derby.log'
    
  """
  #!/usr/bin/env Rscript
  require("ggplot2")
  library(SparkR, lib.loc = c(file.path(Sys.getenv("SPARK_HOME"), "R", "lib")))
  sparkR.session(master = "local[*]", sparkConfig = list(spark.driver.memory = "4g"))
  
  data <- read.df("aggregated", "csv", header="true", inferSchema="true")
  data <- collect(data)

  if (identical("$resultFile","ess")) {
    XXXXX
  } else if (identical("$resultFile", "summaryStatistics")) {
    XXXXX
  } else {
    stop("resultFile not recognized")
  }
    
  ggsave("${resultFile}.pdf", p, width = 20, height = 5, limitsize = FALSE)
  """
}
```

### Several nodes consuming same output (e.g. code, generated data)

```
data.into {
  data1
  data2
}
```
    
## Misc


- To show output add ``echo true``, useful e.g. in combination with adding ``pwd`` to quickly jump to work folders.
- Can import java stuff in nextflow: see first response in https://github.com/nextflow-io/nextflow/issues/238
- Don't forget ``-resume`` option when running to enable use of cache
- Use ``set -e`` in bash script to make it crash if one command crashes and don't forget to use ``Experiment.startAutoExit(..)`` if using the inits package
- note: can use things like ``${Math.pow(dim,refScale)}``


TODO:
- make bins reproducible
    - compiling stuff as tasks not great because of r
    - singularity? 
        - current incubating though..
        - maybe not workeable on westgrid anyways--so maybe after all the best is to use template, local installs, pack racks

> This then defines the work-flow to some extent… Singularity container images must be built and configured on a host where you have root access (this can be a physical system or on a VM or Docker image). Once the container image has been configured it can be used on a system where you do not have root access as long as Singularity has been installed there.
