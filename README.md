# snakemake-on-slurm-clusters
This README describes how to run a snakemake pipeline on a cluster using the [Slurm Workload Manager](https://slurm.schedmd.com/overview.html). Note that this is instruction is targeted specifically for the SLURM cluster setup at my home institute HITS.

With the introduction of Snakemake 8 and the [Snakemake executor plugin for slurm](https://snakemake.github.io/snakemake-plugin-catalog/plugins/executor/slurm.html), the setup got a lot easier compared to the older versions 🙂

To use snakemake on our institute clusters, follow the instructions as provided in the [plugin's documentation](https://snakemake.github.io/snakemake-plugin-catalog/plugins/executor/slurm.html). I just want to note a few things that are special to our clusters: 
- You don't need to provide a Slurm account and you can omit the `slurm_account` argument in the `--default-resources` command line parameters. Snakemake will issue a warning but this can be safely ignored.
- You have to specify the maximum number of jobs to submit at once using `--jobs N`. I'd recommend setting `N` to 100 to not spam the waiting queue.
- Similarly, you can omit the `slurm_partition` argument.
- The runtime limit on our clusters is 24h and setting the `runtime` parameter for a rule to a higher number will have no effect.
- Use the `localrules` parameter with care: it's okay to run small jobs (a few seconds without heavy CPU requirements) on the login node, but anything else MUST be submitted.
- **ALWAYS** make sure your jobs are properly submitted to the cluster and are not running on the login node. If a job was properly submitted, you should see a hint in the log (something like `Job 1 has been submitted with SLURM jobid 1234`). You can also check that you are not hogging ressources on the login node using good old `htop`.
- In case you want to allocade an entire node, you have to ask for all threads a node provides. So e.g. if one node has 16 cores and 2 threads each, you need to set `threads: 32`.
- To get a brief summary of your jobs on the cluster (pending, running, ...), create file called `clusterstatus` in `.local/bin` and paste the following small script:
  ```bash
     #!/bin/bash squeue -u $(whoami) | awk '
     BEGIN {
     abbrev[“R”]=“(Running)“
     abbrev[“PD”]=”(Pending)“
     abbrev[“CG”]=”(Completing)“
     abbrev[“F”]=”(Failed)“
     }
     NR>1 {a[$5]++}
     END {
     for (i in a) {
     printf ”%-2s %-12s %d\n”, i, abbrev[i], a[i]
     }
     }'
  ```
  When you run `clusterstatus`, you will see an output like this: `R (Running) 55` that tells you that you currently have 55 jobs running on the cluster.
- With the new slurm plugin, jobs should be automatically cancelled when you terminate your snakemake pipeline. However, you should always make sure that this the case (e.g. using `clusterstatus`). If jobs are still running or pending, terminate them manually by running `scancel -u $(whoami)`.
- You can use `sinfo` to see the current usage of the cluster, how many nodes are idle, used, or down.
- You can use `squeue` to see the current scheduling queue including the priority of all jobs.
- Always write files to the external file system `/hits/fast` and not to your home directory. 
- Try to separate your jobs into chunks as small as possible and try to allocate as few ressources as possible/reasonable. It is more likely that your jobs will be scheduled soon if you only require a few threads or one node. If you try to do everything at once and ask for 20 nodes, you will have to wait a while for your pipeline to finish. 

<hr>

> [!CAUTION]
> The following description is only applicable to Snakemake version < 8.

## Setting up the environment
Snakemake recommends installing snakemake using conda. To setup the conda environment on your cluster follow these steps: 

1. Create an `environment.yml` file containing all required conda or pip installs. More information on conda's environment files can be found [here](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#sharing-an-environment).
 This file needs to at least contain:

   ``` yml
   channels:
    - bioconda
    - conda-forge
   dependencies:
    - snakemake
    - pip
    - pip:
        - cookiecutter
        - pandas
   ```

2. On your cluster: install the Python3 version of miniconda following the instructions [here](https://conda.io/projects/conda/en/latest/user-guide/install/index.html). If there is already a working conda version available on your system you can skip this step. <br> 
**!** For the next steps I am assuming, that conda is in your PATH variable, if you use a conda version pre-installed on your system, make sure this is the case.

3. Create a conda environment using the `environment.yml` from step 1:
    ```
    conda env create --name demo --file environment.yml
    ```

4. Activate the conda environment:
   ```
   conda activate demo
   ```

## Setting up the slurm profile
To use the power of snakemake on our slurm cluster we first need to create a snakemake profile. Execute all following steps on your cluster.

1. Create a directory for the snakemake profiles and open it:
   ``` 
   mkdir -p ~/.config/snakemake && cd ~/.config/snakemake 
   ```

2. In this directory run 
   ```
   cookiecutter https://github.com/Snakemake-Profiles/slurm.git
   ```
   The first prompt asks you to set a name `profile_name` for the profile. You can either set a custom name leave it empty, in this case the profile will be named `slurm`.
   Leave all other prompts empty for now (just press enter). 

3. In the `~/.config/snakemake` directory should now be a folder called `profile_name` containing
   
   ```
   config.yaml
   slurm-jobscript.sh
   slurm-status.py
   slurm-submit.py
   slurm_utils.py  
   ```

4. Open this directory and open the `config.yaml` file. In this file you can set any command line options that you would normally add to the `snakemake` call. [Here](https://snakemake.readthedocs.io/en/stable/executing/cli.html#all-options) is a list of available options. There are a few default options set already as example. You can change them or add some if you wish.<br>
   **Important:** To use the installed snakemake profile scripts make sure to not change or delete the following lines:

   ``` yaml
   jobscript: "slurm-jobscript.sh"
   cluster: "slurm-submit.py"
   ```

   If you used conda to install other packages than snakemake you should also add:

   ``` yaml
   use-conda: True
   ```

   A configuration file might look like this.

   ```yaml
   jobscript: "slurm-jobscript.sh"
   cluster: "slurm-submit.py"
   cluster-status: "slurm-status.py" 
   jobs: 20 # max number of jobs to submit at once
   use-conda: True # run job in conda environment
   restart-times: 3 # restart failed jobs three times
   max-jobs-per-second: 1 # max number of cluster jobs per second
   latency-wait: 60 # wait 60 seconds after job is finished if output is not present
   rerun-incomplete: True # rerun all jobs with incomplete output
   printshellcmds: True # print the shell commands that will be executed
   ```

5. Now let's define some default resource utilization. For this create a file `cluster_config.yml` in the current directory (`~/.config/snakemake/profile_name`). In this file add the default arguments that should be passed to sbatch. Define them using the following format and use correct sbatch expressions. See [the slurm documentation](https://slurm.schedmd.com/sbatch.html#lbAG) for a list of available options.

   ``` yaml
   __default__:
       option1: value1
       option2: value2
       ...
       optionN: valueN
   ```

   A cluster config file might look like this: 
   ```yaml
   __default__:
       account: your_account
       partition: partition # name of the partition to use
       time: 60 # default run time in minutes
       mem: 1GB # default memory
       nodes: 1 
       cpus-per-task: 1 
       threads-per-core: 1
       output: where/to/write/output.txt
       error: where/to/write/error.txt
       mail-type: FAIL
       mail-user: your-email@adress.com
    ```

6. Open ```slurm-submit.py``` and edit the ```CLUSTER_CONFIG``` constant so it points to your ```cluster_config.yml``` file.

7. Snakemake will run every rule with these options. Some of your rules might however need more or less resources than these default values. You can specify these values using the `resources`-keyword in these rules, for example:

   ``` python
   rule:
       input:     ...
       output:    ...
       resources:
           mem=10000, # use 10GB of memory
           runtime=360, # run for 6 hours
       shell:
           "..."
   ```
   If we want to run this rule on multiple nodes in a cluster, things get a little more complicated. 
   The following example will run an `OpenMPI` program on 2 nodes with 16 cores each and will allocate the entire node.
   
   ``` python
   rule:
      input:     ...
      output:    ...
      threads: 32  # #thraeds-per-core * #cores-per-node, see details below
      resources:
         mem=10000, # use 10GB of memory
         runtime=360, # run for 6 hours
         nodes=2, # use two nodes
         cpus_per_task=16,
         slurm_extra="-B 2:8:1 --ntasks-per-node 1 --hint compute_bound",  # some additional options for slurm, check your cluster manual to see what is necessary for your setup
      shell:
         "mpirun ..."
   ```
   To allocate an entire node, you have to ask for all threads each node provides. So if one node has 16 cores and 2 threads each, set `threads: 32`. Don't change this number when increasing the number of nodes.

   For more information on rule specific resources and sbatch options see the [snakemake documentation](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#resources).

8. Finally, we can run the snakemake pipeline. Open the folder containing your `Snakefile`, open a `tmux` session and run
   
   ```
   snakemake --profile profile_name
   ```

   Snakemake will submit each rule as a separate sbatch job to the submission system. The snakemake process itself will however run on your login node. As this process is mostly waiting this should be fine. 

   If you do not want to use `tmux` to run snakemake for whatever reason make sure to keep your ssh connection alive, otherwise the snakemake process will be terminated. 
   
   **Note:** Terminating the snakemake process will **not** terminate already submitted jobs.  
