======================================
Setting up a Solent NEMO configuration
======================================

URL:: http://nemo-reloc.readthedocs.io/en/latest/Solent.html

Issues that arose
=================

* ...


Follow PyNEMO recipe for Lighthouse Reef: ``http://pynemo.readthedocs.io/en/latest/examples.html#example-2-lighthouse-reef``
Follow PyNEMO recipe for Lighthouse Reef: ``http://nemo-reloc.readthedocs.io/en/latest/SEAsia.html``
Follow PyNEMO recipe for Liverpool Bay: ``http://nemo-reloc.readthedocs.io/en/latest/LBay.html``

----

Recipe Notes
============

Define working directory and other useful shortcuts::

  export WDIR=/work/n01/n01/jelt/Solent/
  export INPUTS=/work/n01/n01/jelt/lighthousereef/INPUTS
  export CDIR=$WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/CONFIG
  export TDIR=$WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/TOOLS

Load modules::

  module swap PrgEnv-cray PrgEnv-intel
  module load cray-netcdf-hdf5parallel
  module load cray-hdf5-parallel

Follow recipe. For LHReef the INPUT files were all prepared. Now they are not
so make them as and when they are required. Ideally I build all those input
files without having to build opa and xios. However the grid tools are in the
nemo checkout so it is probably easier to just setup xios and compile nemo
rather than try and abstract the grid tools::

I will be more concise about what is going on here. For a more expansive walk
though see the Liverpool Bay notes::

  mkdir $WDIR
  cd $WDIR
  mkdir INPUTS

  svn co http://forge.ipsl.jussieu.fr/nemo/svn/branches/2014/dev_r4621_NOC4_BDY_VERT_INTERP@5709
  svn co http://forge.ipsl.jussieu.fr/ioserver/svn/XIOS/branchs/xios-1.0@629

Need to get arch files from INPUTS::

  cd $WDIR/xios-1.0
  cp $INPUTS/arch-XC30_ARCHER.* ./arch

Implement make command::

  ./make_xios --full --prod --arch XC30_ARCHER --netcdf_lib netcdf4_par

Step 2. Obtain and apply patches::

  cd $CDIR/../NEMO/OPA_SRC/SBC
  patch -b < $INPUTS/fldread.patch
  cd ../DOM
  patch -b < $INPUTS/dommsk.patch
  cd ../BDY
  patch -b < $INPUTS/bdyini.patch
  cd $CDIR
  rm $CDIR/../NEMO/OPA_SRC/TRD/trdmod.F90
  cp $INPUTS/arch-* ../ARCH

On first make only choose OPA_SRC::

  ./makenemo -n Solent -m XC_ARCHER_INTEL -j 10

It breaks. Remove key_lim2 from cpp*fcm file and remake::

  vi LBay/cpp_Solent.fcm
  ...

Copy some input files to new configuration path::

  cp $INPUTS/cpp_LH_REEF.fcm ./Solent/cpp_Solent.fcm
  cp $INPUTS/dtatsd.F90 Solent/MY_SRC/

Remake (ignoring the vi edit)::

  ./makenemo -n Solent -m XC_ARCHER_INTEL -j 10


Build tools to create configuration inputs
------------------------------------------

To generate bathymetry, initial conditions and grid information we first need
to compile some of the NEMO TOOLS (after a small bugfix - and to allow direct
passing of arguments). For some reason GRIDGEN doesn’t like INTEL::

  cd $WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/TOOLS/WEIGHTS/src
  patch -b < $INPUTS/scripinterp_mod.patch
  patch -b < $INPUTS/scripinterp.patch
  patch -b < $INPUTS/scrip.patch
  patch -b < $INPUTS/scripshape.patch
  patch -b < $INPUTS/scripgrid.patch

  cd ../../
  ./maketools -n WEIGHTS -m XC_ARCHER_INTEL
  ./maketools -n REBUILD_NEMO -m XC_ARCHER_INTEL

  module unload cray-netcdf-hdf5parallel cray-hdf5-parallel
  module swap PrgEnv-intel PrgEnv-cray
  module load cray-netcdf cray-hdf5
  ./maketools -n GRIDGEN -m XC_ARCHER

  module swap PrgEnv-cray PrgEnv-intel

----

1. Generate new coordinates file
++++++++++++++++++++++++++++++++

Generate a ``coordinates.nc`` file from a parent NEMO grid at some resolution.
**Plan:** Use tool ``create_coordinates.exe`` which reads cutting indices and
parent grid location from ``namelist.input`` and outputs a new files with new
resolution grid elements.

First we need to figure out the indices for the new domain, from the parent grid.
Move parent grid into INPUTS::

  cp $INPUTS/coordinates_ORCA_R12.nc $WDIR/INPUTS/.

Inspect this parent coordinates file to define the boundary indices for the new config.

Note, I used FERRET locally::

  $livljobs2$ scp jelt@login.archer.ac.uk:/work/n01/n01/jelt/LBay/INPUTS/coordinates_ORCA_R12.nc ~/Desktop/.
  ferret etc

  NW: 50.84, -2.0
  SE: 50.30, -0.7
  shade/i=3409:3423/j=2202:2210 NAV_LAT
  shade/i=3409:3423/j=2202:2210 NAV_LON


Copy namelist file from LH_reef and edit with new indices, retaining use of
ORCA_R12 as course
parent grid::

  cd $TDIR/GRIDGEN
  cp $INPUTS/namelist_R12 ./
  vi namelist_R12
  ...
  cn_parent_coordinate_file = '../../../../INPUTS/coordinates_ORCA_R12.nc'
  ...
  nn_imin = 3409
  nn_imax = 3423
  nn_jmin = 2202
  nn_jmax = 2210
  nn_rhox  = 7
  nn_rhoy = 7

  ln -s namelist_R12 namelist.input
  ./create_coordinates.exe
  cp 1_coordinates_ORCA_R12.nc $WDIR/INPUTS/coordinates.nc

This creates a coordinates.nc file with contents, which are now copied to
INPUTS::

Now we need to generate a bathymetry on this new grid.



2. Generate bathymetry file
+++++++++++++++++++++++++++

Grab some AMM60 bathymetry data::

  cp /work/n01/n01/kariho40/NEMO/GRID/bathyfile_AMM60_nosmooth.nc /work/n01/n01/jelt/Solent/INPUTS/.

Copy namelist for reshaping GEBCO data. Rename it::

  cp $INPUTS/namelist_reshape_bilin_gebco $WDIR/INPUTS/namelist_reshape_bilin_amm60

Edit namelist to point to correct input file. Edit lat and lon variable names to
 make sure they match the nc file content (used e.g.
``ncdump -h bathyfile_AMM60_nosmooth.nc`` to get input
variable names). Also changed ``gebco`` tails to ``amm60``::

  vi $WDIR/INPUTS/namelist_reshape_bilin_amm60
  ...
  &grid_inputs
      input_file = 'bathyfile_AMM60_nosmooth.nc'
      nemo_file = 'coordinates.nc'
      datagrid_file = 'remap_data_grid_amm60.nc'
      nemogrid_file = 'remap_nemo_grid_amm60.nc'
      method = 'regular'
      input_lon = 'lon'
      input_lat = 'lat'
      nemo_lon = 'glamt'
      nemo_lat = 'gphit'
    ...

    &interp_inputs
      input_file = "bathyfile_AMM60_nosmooth.nc"
      interp_file = "data_nemo_bilin_amm60.nc"
      input_name = "Bathymetry"
    ...

    &shape_inputs
      interp_file = 'data_nemo_bilin_amm60.nc'
      output_file = 'weights_bilinear_amm60.nc'
    ...

Do some things to 1) flatten out land elevations, 2) make depths positive. *(James
noted a problem with the default nco module)*::

  cd $WDIR/INPUTS
  module load nco/4.5.0
  ncap2 -s 'where(Bathymetry > 0) Bathymetry=0' bathyfile_AMM60_nosmooth.nc tmp.nc
  ncflint --fix_rec_crd -w -1.0,0.0 tmp.nc tmp.nc bathyfile_AMM60_nosmooth.nc
  rm tmp.nc


Restore the original parallel modules, which were removed to fix tool building issue::

  module unload nco cray-netcdf cray-hdf5
  module load cray-netcdf-hdf5parallel cray-hdf5-parallel

Execute first scrip things::

  $TDIR/WEIGHTS/scripgrid.exe namelist_reshape_bilin_amm60
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bilin_amm60
  $TDIR/WEIGHTS/scripinterp.exe namelist_reshape_bilin_amm60

Output files::

  bathy_meter.nc



3. Generate initial conditions
++++++++++++++++++++++++++++++


Skip building SOSIE as it is already done.
Proceed with Step 6::

  export PATH=~/local/bin:$PATH
  cd $WDIR/INPUTS

Obtain the fields to interpolate. Use AMM60 data::

#  cp $INPUTS/initcd_votemper.namelist .
#  cp $INPUTS/initcd_vosaline.namelist .

Actually copy these namelists from the LBay configuration::

  cp $WDIR/../LBay/INPUTS/initcd_votemper.namelist .
  cp $WDIR/../LBay/INPUTS/initcd_vosaline.namelist .

Find the AMM60 indices using FERRET on the bathy_meter.nc file: ``shade log(Bathymetry[I=540:750, J=520:820])``

Note that the temperature and salinity variables are ``thetao`` and ``so``

::

  module unload cray-netcdf-hdf5parallel cray-hdf5-parallel
  module load cray-netcdf cray-hdf5
  module load nco/4.5.0

  cd $WDIR/INPUTS

  ncks -d x,645,710 -d y,530,600 /work/n01/n01/kariho40/NEMO/NEMOGCM_jdha/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/CONFIG/AMM60smago/EXP_notradiff/OUTPUT/AMM60_5d_20131013_20131129_grid_T.nc $WDIR/INPUTS/cut_down_20131013_Solent_grid_T.nc

  module unload nco cray-netcdf cray-hdf5
  module load cray-netcdf-hdf5parallel cray-hdf5-parallel



Edit namelists, switch ``LBay`` for ``Solent``::

  vi initcd_votemper.namelist
  ...
  ctarget  = 'Solent'

  vi initcd_vosaline.namelist
  ...
  ctarget  = 'Solent'


Do stuff. I think the intention was for SOSIE to flood fill the land::

  sosie.x -f initcd_votemper.namelist

Creates::

  sosie_mapping_AMM60-Solent.nc
  thetao_AMM60-Solent_2013.nc4

Repeat for salinity::

  sosie.x -f initcd_vosaline.namelist

Creates::

  so_AMM60-Solent_2013.nc4


Now do interpolation as before. First copy the namelists::

  cp $INPUTS/namelist_reshape_bilin_initcd_votemper $WDIR/INPUTS/.
  cp $INPUTS/namelist_reshape_bilin_initcd_vosaline $WDIR/INPUTS/.

Edit the input files::

  vi $WDIR/INPUTS/namelist_reshape_bilin_initcd_votemper
  &grid_inputs
    input_file = 'thetao_AMM60-Solent_2013.nc4'
  ...

  &interp_inputs
    input_file = "thetao_AMM60-Solent_2013.nc4"
    input_name = "thetao"
  ...

Simiarly for the *vosaline.nc file::

  vi $WDIR/INPUTS/namelist_reshape_bilin_initcd_vosaline
  &grid_inputs
    input_file = 'so_AMM60-Solent_2013.nc4'
  ...

  &interp_inputs
    input_file = "so_AMM60-Solent_2013.nc4"
    input_name = "so"
  ...


Produce the remap files::

  $TDIR/WEIGHTS/scripgrid.exe namelist_reshape_bilin_initcd_votemper

Creates ``remap_nemo_grid_R12.nc`` and ``remap_data_grid_R12.nc``. Then::

**This breaks with a seg fault**

  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bilin_initcd_votemper

To do::

  $TDIR/WEIGHTS/scripinterp.exe namelist_reshape_bilin_initcd_votemper
  $TDIR/WEIGHTS/scripinterp.exe namelist_reshape_bilin_initcd_vosaline

4. Generate weights for atm forcing
+++++++++++++++++++++++++++++++++++

Obtain namelist files and data file::

  cp $INPUTS/namelist_reshape_bilin_atmos $WDIR/INPUTS/.
  cp $INPUTS/namelist_reshape_bicubic_atmos $WDIR/INPUTS/.

Generate cut down drowned precip file (note that the nco tools don't like the
parallel modules)::

  module unload cray-netcdf-hdf5parallel cray-hdf5-parallel
  module load cray-netcdf cray-hdf5
  ncks -d lon,265.,280. -d lat,15.,25. /work/n01/n01/acc/ORCA0083/NEMOGCM/CONFIG/R12_ORCA/EXP00/FORCING/drowned_precip_DFS5.1.1_y1979.nc $WDIR/INPUTS/cutdown_drowned_precip_DFS5.1.1_y1979.nc

Setup weights files for the atmospheric forcing::

  $TDIR/WEIGHTS/scripgrid.exe namelist_reshape_bilin_atmos
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bilin_atmos
  $TDIR/WEIGHTS/scripshape.exe namelist_reshape_bilin_atmos
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bicubic_atmos
  $TDIR/WEIGHTS/scripshape.exe namelist_reshape_bicubic_atmos

5. Generate mesh and mask files for open boundary conditions
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

6. Generate boundary conditions with PyNEMO
+++++++++++++++++++++++++++++++++++++++++++

----
