# Generating Instaseis Databases on Tiger

This notebook provides a practical guide to compiling AxiSEM on Tiger for the purpose
of generating reciprocal Green's function databases for use with Instaseis. For a more
comprehensive guide to using AxiSEM, consult the user manual which can be found 
[here](https://geodynamics.org/cig/software/axisem/axisem-manual.pdf).

## Setting up your Environment

Begin by logging into Tiger, and loading the following modules by executing:


```
module load openmpi/intel-16.0/1.10.2/64 hdf5/intel-16.0/openmpi-1.10.2/1.8.16 
netcdf/intel-16.0/hdf5-1.8.16/openmpi-1.10.2/4.4.0 intel/16.0/64/16.0.4.258
```

To check this has been executed properly, run the command `module list`. The output should
display:

<pre>
Currently Loaded Modulefiles:
  1) openmpi/intel-16.0/1.10.2/64                         4) intel-mkl/11.3.4/4/64
  2) hdf5/intel-16.0/openmpi-1.10.2/1.8.16                5) intel/16.0/64/16.0.4.258
  3) netcdf/intel-16.0/hdf5-1.8.16/openmpi-1.10.2/4.4.0
</pre>

**Note: The order in which the modules are loaded affects how the code is compiled, 
so it is critical your output matches what is displayed above before continuing.**

Next, we need to create a link to the Intel compiler, Open MPI, NetCDF, and HDF5 libraries. Open 
your `~/.bashrc` file and add the following lines:

<pre>
LD_LIBRARY_PATH=/opt/intel/compilers_and_libraries_2016.4.258/linux/compiler/lib/intel64_lin:/usr/local
/openmpi/1.10.2/intel160/x86_64/lib64:/usr/local/hdf5/intel-16.0/openmpi-1.10.2/1.8.16/lib64:/usr/local
/netcdf/intel-16.0/hdf5-1.8.16/openmpi-1.10.2/4.4.0/lib64

export LD_LIBRARY_PATH
</pre> 

then execute `source ~/.bashrc`.

## Compiling and Running AxiSEM

Next, navigate to your directory containing AxiSEM within the `/scratch/gpfs/$NETID` directory 
where `$NETID` is your Princeton NetID. Once there, open the file `make_axisem.macros` and edit 
the entries so they correspond to those shown below:

<pre>
MPIRUN = mpirun

USE_NETCDF = true
USE_PAR_NETCDF = true
NETCDF_PATH = /usr/local/netcdf/intel-16.0/hdf5-1.8.16/openmpi-1.10.2/4.4.0

SERIAL = false

INCLUDE_MPI = false

# GFORTRAN (debug)
CC         = mpicc
FC         = mpif90
FFLAGS     = -g -fbacktrace -fbounds-check -frange-check -pedantic
CFLAGS     = -g -fbacktrace -Wall
LDFLAGS    = -g -fbacktrace -Wall
</pre>

At this point, you can navigate to the `MESHER` directory and adjust the parameters in
the file `inparam_mesh` to be suitable for your desired application. The mesher can be
run by executing `./submit.csh`. Check the progress of the mesher with `tail -f OUTPUT`
and once `....DONE WITH MESHER !` appears you can move the mesh and accompanying files 
to the `SOLVER` directory by executing `./movemesh.csh $MESHNAME` where `$MESHNAME` is 
chosen by the user.

After successfully moving the mesh, navigate to the `SOLVER` directory
and open the file `inparam_basic`, setting the first entry to:

<pre>
SIMULATION_TYPE   force
</pre>

and adjusting the additional parameters as you see fit. Next, open the file `inparam_advanced`
and set the following parameters:

<pre>
USE_NETCDF           true
DEFLATE_LEVEL        0
KERNEL_WAVEFIELDS    true
</pre>

In addition, make sure that sensible values are chosen for the following two parameters
*(whose default values are for an unknown, larger than Earth sized planet... probably for
use with Paraview!)*:
<pre>
KERNEL_RMIN
KERNEL_RMAX
</pre>

You can also change the parameters in `inparam_source` and `inparam_hetero` to suit your needs.

The final step before running the code is opening the file `submit.csh` and changing the 
parameters under the `slurm` section according to the **Submitting a Job** section of the 
[Tiger Tutorial](https://researchcomputing.princeton.edu/computational-hardware/tiger/tutorials).

Once this is complete, you're finally ready for the exciting part - simulating wave propagation!
You can begin your simulation by executing `./submit.csh $DIRNAME -q slurm` where
`$DIRNAME` is the name of the output directory. 

## Postprocessing and Repacking Databases

Once your simulation has finished running, navigate to the run directory and execute 
`./field_transform.sh`. At this point the databases have been created, and are stored in 
`P?/Data/ordered_output.nc4`. The files are likely very large, so you'll want to move them
off of the cluster to a drive for storage. For the remainder of the tutorial it is assumed
that the user is on a machine that has [Instaseis](http://instaseis.net/#) installed and 
has worked through the accompanying tutorial.

Create a directory to store the databases in, and within this directory make two new directories,
`PX` and `PZ`. Move the corresponding `ordered_output.nc4` files from the vertical and horizontal
simulations into these directories. Then, execute:

<pre>
python -m instaseis.scripts.repack_db --method repack $INPUT_DIR $OUTPUT_DIR
</pre>

where `$INPUT_DIR` is the directory containing the `PX` and `PZ` directories. The repacked
databases should now be ready to use with Instaseis for generating synthetic seismograms.

Congratulations, you've successfully created your own Instaseis database!
