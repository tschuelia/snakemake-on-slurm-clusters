# snakemake-on-slurm-clusters
This README describes how to run a snakemake pipeline on a cluster using the [Slurm Workload Manager](https://slurm.schedmd.com/overview.html).

## Setting up the environment
Snakemake recommends installing snakemake using conda. To setup the conda environment on your cluster follow these steps: 

1. Create an `environment.yml` file containing all required conda or pip installs. More information on conda's environment files can be found [here](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#sharing-an-environment).
 This file needs to at least contain:

   ``` yml
   channels:
    - bioconda
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

5. Now lets define the resource utilization. For this create a file `cluster_config.yml` in the current directory (`~/.config/snakemake/profile_name`). In this file add the default arguments that should be passed to sbatch. Define them using the following format and use correct sbatch expressions. See [the slurm documentation](https://slurm.schedmd.com/sbatch.html#lbAG) for a list of available options.

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
           nodes=4, # use 4 nodese
           mem=10000, # use 10GB of memory
           time=360, # run for 6 hours
       shell:
           "..."
   ```

   For more information on rule specific resources and sbatch options see the [snakemake documentation](https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html#resources). 

8. Finally we can run the snakemake pipeline. Open the folder containing your `Snakefile`, open a `tmux` session and run
   
   ```
   snakemake --profile profile_name
   ```

   Snakemake will submit each rule as a separate sbatch job to the submission system. The snakemake process itself will however run on your login node. As this process is mostly waiting this should be fine. 

   If you do not want to use `tmux` to run snakemake for whatever reason make sure to keep your ssh connection alive, otherwise the snakemake process will be terminated. 
   
   **Note:** Terminating the snakemake process will **not** terminate already submitted jobs.  