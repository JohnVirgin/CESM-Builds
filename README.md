# CESM-Builds
 Source Code and Use Instructions for CESM on Scinet's Niagara

# Source Code and Picking a Particular Version
I'm just going to be discussing running/modifying the model version that's already sitting in my home directory, as it's been modified accordingly for Niagara specifically.

### Pre-configured version from my home directory (CESM2.1.0)
Log into scinet and ```cd``` over to my home directory:

```
cd /home/c/cgf/jgvirgin
```
You should see a directory labelled ```cesm2_1_0```, which is the version of CESM2 that's configured for use on Niagara. Copy that directory over to your home directory. If you don't have permissions, see the alternative below

### Getting the Source code from NCAR and the Niagara specific configuration files from this github
If permissions aren't allowing you to access the ```cesm2_1_0``` directory from my home folder, you can download the model source code from NCAR directly:

```
git clone -b release-cesm2.1.0 https://github.com/ESCOMP/cesm.git
```

move into the ```cesm``` folder that was just downloaded and run the checkout externals command to get the component specific source code (land, atmosphere, sea ice, ocean, etc). Make sure subversion is loaded before checkout (```module load subversion```)

```
cd cesm
./manage_externals/checkout_externals
```

Now You've got a full copy of the model (minus the input data). All that's left is to access the Niagara specific .xml files in this repo and replace the ones sitting in your ```cesm2_1_0``` directory.

```
git clone https://github.com/JohnVirgin/CESM-Builds/Niagara/cesm2_1_0_xml
```

These three .xml files- config_compilers, config_machines, and config_batch- configure CESM's compilers, hardware specifics, and SLURM batch submission system specifically for Niagara. Three files of the same names should be sitting in:

```
~/cesm2_1_0/cime/config/cesm/machines
```

Replace those default files with the ones from this repo, and you should be ready to go.

# Running the Model (Quickstart version)
Running the model consists of four mandatory steps, with an number of options in between to modify a given case with your own desired specifications. Steps as follows are:

```
./create_newcase
./case.setup
./case.build
./case.submit
```

### Creating a new case

```cd``` into ```cesm2_1_0/cime/scripts/``` and you'll see an executable file, ```create_newcase```. Execute this with some parameters to initiate a particular CESM experiment. A few mandatory options:

```
--case
```

Specific location of where your case will be created. I have two folders for this, one in my home directory and one in my scratch directory. Regardless of where you put it, the .xml files we're using are configured to automatically file all compiling, archiving, and setup output into your scratch directory. I suggest putting newly created cases in your scratch directory for now, because submitted batch jobs using SLURM refuse to write into your home directory (home directory is read only). You might get some issues if a log file is created during your run and it happens to be sitting in your ```$HOME```.

Do all your building, compiling, running, and everything else out of scratch. When you're all finished, move the case directory you made back to your home to prevent it from being deleted due to inactivity.

```
--res
```

Specifies the resolution of the components being used in your build. What goes into this command depends on what component set you're picking. You can find a list of supported grids for CESM here: http://www.cesm.ucar.edu/models/cesm2/config/compsets.html.

```
--compset
```

Specifies the components going into your build. This option is actually typically used as a shortcut. Compsets that are more common have 'short names' that'll save you from typing out which specific CAM, CLM, POP, etc models you'd like to include here. You can find a list of supported CESM2 compsets here: http://www.cesm.ucar.edu/models/cesm2/cesm/compsets.html. Some component sets haven't been scientifically validated by NCAR yet; get around this by adding ```--run-unsupported``` to the end of the line.

```
--machine
```

Specifies the machine you're running on so CESM knows where to get its modules, compilers, and figure out what batch system we're working with. For us, this is always ```niagara```.

Here's a quick example of a test case I setup and ran:

```
./create_newcase --case $SCRATCH/CESM_Cases/b.e21.B1850.f19_g17.JGV.TestCase.1 --res f19_g17 --compset B1850 --machine niagara
```
The nomenclature I used for the case directory name follows the accepted conventions from NCAR's website: http://www.cesm.ucar.edu/models/cesm1.0/casename_conventions_cesm.html

### Pre Case Setup
Prior to setting up your case after its creation, you'll want to make any load balancing (modifying node and CPU allocations to individual model componenets) changes, or any changes to the model source code. Modification of nodes and CPU allocation to individual model components is done within ```env_mach_pes.xml``` file. Additionally, making modifications to the individual CESM components is done in the ```SourceMods/``` directory. 

### Setup the new case
Head on into the case directory you just made and take a look at all the newly created files. The next step is to setup your case. Execute this option to set up your run directory (which should be in your ```$SCRATCH```). It will also give you a bunch of user-changeable namelist files, where you can specify the modified and new variables that you want the model to output. There's a single namelist file for each model component, and they have naming syntax like ```user_nl_xxx```. Where the `xxx` is replaced with a given model. Lastly, this command also creates hidden files that specify batch information for SLURM You can make your changes to the .xml files we're about to talk about either before or after you run the case setup.

### Build the new case
A lot of case-specific changes will be made just before you build your case:

- [ ] All additions to your namelist variables - ```user_nl_$model```, will be made here. I'll include more information on modifying these below, but it's not important for the (not-so) quick start version.
- [ ] Changes to batch submission options.
- [ ] Changes to run length and run history time-steps.
- [ ] Changes to physics, chemistry, and parameterization schemes.

I won't dive into a lot of these right now, but a few important things to note:
- A lot of the batch submission, run length, and run options are configured in the .xml files that are sitting in your case directory. Now, you can go into each one and manually change things. But NCAR advises against that because the chance of error goes up. Their work around is to include to executable files to make your life easier- ```./xmlchange``` and ```./xmlquery```.
- You can ```./xmlquery``` a variable in ANY .xml file in the directory, and it'll return its value/string.
- You can ```./xmlchange``` any of these variables by specifying the name and what you're changing it to.

There are a few variables you'll find you'll be changing pretty much any time you run the model, listed below:
- STOP_OPTION: Indicates timestep for restart files. Alternatively defined, It's the time units of your run length. This variable is located within ```env_run.xml```
  - ```ndays```,```nmonths```,```nyears```.
- STOP_N: Integer for the number of STOP_OPTION steps to run the model for. Default for this is ```5```, with the default STOP_OPTION is ```ndays```. As such, Running the model from default only takes 5 days. This variable is located in ```env_run.xml```
  - ```int```
- JOB_WALLCLOCK_TIME: Indicates how long your run will compute on the nodes you've allocated. The model will default to the max amount of time allowed on Niagara, which is 24 hours. This'll throw you a warning, though. Format for the string is ```Hours:Minutes:Seconds```. This variable is located in ```env_batch.xml```.
  - ```H:M:S``` (e.g.```5:00:00``` is 5 hours of runtime)
- RESUBMIT: Sets the number of times to resubmit the run. Since Niagara only allows 24 hours per job, you have to set the model to resubmit and initialize from a restart file whenever the job's allocated time ends.
  - ```int```
- CONTINUE_RUN: This is a boolean value that indicates whether or not to continue the run from some initialized case that you never finished. Note that if you start a run from the beginning and set the resubmit to some arbitrary number, ```CONTINUE_RUN``` will automatically swap from ```FALSE``` to ```TRUE``` after the first resubmission.
  - ```FALSE``` or ```TRUE```
- RUN_TYPE: Indicates the type of run being executed. Options reflect running from scratch and running from some reference point. The differences and guides for each of these run types can be found in the Basic_Mods slide deck in the NCAR tutorial.
  - ```startup```, ```hybrid```, and ```branch```

There's many more; open the the ```env``` files to get a full look. One other thing to note: The default model output is set to spit out monthly data, so if you run a case from the default configuration (5 days), it won't give you any model output. If you want it to give you daily data, you need to do this on a per variable basis through the namelist files.

time to run  ```./case.build``` !

This will take a little while to compile the model. You can see the time elapse for each component as it builds.


### Submitting the new case
Once you're ready to submit, you can execute the ```./case.submit``` command, for which it'll give you all the SLURM batch specifics. I'm fairly certain that this executable file is just a wrapper for the hidden file ```.case.run```, which is the actual shell script that submits via slurm. If you want to make changes to the number of nodes you're using or the number of tasks per node, you can do it in here.

IMPORTANT NOTE: The other important thing ```./case.submit``` does is checks to see if you have the necessary input data to run your case (forcings, initial conditions, etc.). You'll see it check for this when you run ```./case.submit```, and if you don't have it, it'll download the necessary data from NCAR.

In the ```config_machines.xml``` file, the input data directory is hard coded to my $SCRATCH (```/scratch/c/cgf/jgvirgin/cesm2_1_0/inputdata/```). I did this so we wouldn't all be downloading input data for similar cases. This makes no difference for when you try to run cases (or at least it shouldn't, but I still need to check permissions), but at least you know where the input data is being downloaded to.

# Adding Output Variables and Changing the Archive Files
To be updated in the future
