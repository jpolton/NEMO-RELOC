=======================================
Setting up a SE Asia NEMO configuration
=======================================

Issues that arose
=================

* Helpful to add in extra module command for first step
* Fix the sosie make.macro command
* Queue accound is now n01-NOCL in runscript

Following the recipe
====================

Follow PyNEMO recipe for Lighthouse Reef: ``http://pynemo.readthedocs.io/en/latest/examples.html#example-2-lighthouse-reef``
Define working directory::

  export WDIR=/home/n01/n01/jelt/work/lighthousereef/

Load modules::

  module swap PrgEnv-cray PrgEnv-intel
  module load cray-netcdf-hdf5parallel
  module load cray-hdf5-parallel

**James: You should add the above 3 lines as an e.g. in the docs**. The *swap* one threw me for a while.

Follow recipe. Step 1::

  cd $WDIR
  mkdir INPUTS
  cd INPUTS
  wget ftp.nerc-liv.ac.uk:/pub/general/jdha/inputs.tar.gz
  tar xvfz inputs.tar.gz
  rm inputs.tar.gz
  cd ../
  svn co http://forge.ipsl.jussieu.fr/nemo/svn/branches/2014/dev_r4621_NOC4_BDY_VERT_INTERP@5709
  svn co http://forge.ipsl.jussieu.fr/ioserver/svn/XIOS/branchs/xios-1.0@629
  cd xios-1.0
  cp $WDIR/INPUTS/arch-XC30_ARCHER.* ./arch

Implement make command::

  ./make_xios --full --prod --arch XC30_ARCHER --netcdf_lib netcdf4_par

The comile works. But there are lots of warnings of three types::

  WARNING: /fs4/n01/n01/jelt/lighthousereef/xios-1.0/bld.cfg: LINE 40:
           ${PWD}: cyclic dependency, variable not expanded, ${PWD}: cyclic dependency, variable not expanded, ${PWD}: cyclic dependency, variable not expanded

  WARNING: CFLAGS__date.flags: duplicated targets for building:

  /home/n01/n01/jelt/work/lighthousereef/xios-1.0/Makefile:1418: warning: ignoring old commands for target `FPPKEYS.flags'


Step 2, as far as the ``makenemo`` call::

  export CDIR=$WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/CONFIG
  export TDIR=$WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/TOOLS
  cd $CDIR/../NEMO/OPA_SRC/SBC
  patch -b < $WDIR/INPUTS/fldread.patch
  cd ../DOM
  patch -b < $WDIR/INPUTS/dommsk.patch
  cd ../BDY
  patch -b < $WDIR/INPUTS/bdyini.patch
  cd $CDIR
  rm $CDIR/../NEMO/OPA_SRC/TRD/trdmod.F90
  cp $WDIR/INPUTS/arch-* ../ARCH
  ./makenemo -n LH_REEF -m XC_ARCHER_INTEL -j 10

Build fails (as described) so remove ``key_lim2`` (as described) and re-issue the make command::

  ./makenemo -n LH_REEF -m XC_ARCHER_INTEL -j 10

  cp $WDIR/INPUTS/cpp_LH_REEF.fcm ./LH_REEF
  cp $WDIR/INPUTS/dtatsd.F90 LH_REEF/MY_SRC/

Step 3 (seemed to work OK)::

  cd $WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/TOOLS/WEIGHTS/src
  patch -b < $WDIR/INPUTS/scripinterp_mod.patch
  patch -b < $WDIR/INPUTS/scripinterp.patch
  patch -b < $WDIR/INPUTS/scrip.patch
  patch -b < $WDIR/INPUTS/scripshape.patch
  patch -b < $WDIR/INPUTS/scripgrid.patch
  cd ../../
  ./maketools -n WEIGHTS -m XC_ARCHER_INTEL
  ./maketools -n REBUILD_NEMO -m XC_ARCHER_INTEL
  module unload cray-netcdf-hdf5parallel cray-hdf5-parallel
  module swap PrgEnv-intel PrgEnv-cray
  module load cray-netcdf cray-hdf5
  ./maketools -n GRIDGEN -m XC_ARCHER
  module swap PrgEnv-cray PrgEnv-intel
  export TDIR=$WDIR/dev_r4621_NOC4_BDY_VERT_INTERP/NEMOGCM/TOOLS

Step 4::

  cd $TDIR/GRIDGEN
  cp $WDIR/INPUTS/namelist_R12 ./
  ln -s namelist_R12 namelist.input
  ./create_coordinates.exe
  cp 1_coordinates_ORCA_R12.nc $WDIR/INPUTS/coordinates.nc

Step 5::

  cd $WDIR/INPUTS
  module load nco/4.5.0
  ncap2 -s 'where(topo > 0) topo=0' gebco_1_cutdown.nc tmp.nc
  ncflint --fix_rec_crd -w -1.0,0.0 tmp.nc tmp.nc gebco_in.nc
  rm tmp.nc
  module unload nco cray-netcdf cray-hdf5
  module load cray-netcdf-hdf5parallel cray-hdf5-parallel
  $TDIR/WEIGHTS/scripgrid.exe namelist_reshape_bilin_gebco
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bilin_gebco
  $TDIR/WEIGHTS/scripinterp.exe namelist_reshape_bilin_gebco


Step 6 didn't find the ``make.macro`` file. Copy it from ``jdha`` and edit the path::

  cp /home/n01/n01/jdha/sosie/make.macro /home/n01/n01/jelt/sosie/.

  vi /home/n01/n01/jelt/sosie/make.macro
  # Directory to install binaries:
  INSTALL_DIR = /home/n01/n01/jelt/local

Proceed with Step 6::

  cd ~
  mkdir local
  svn co svn://svn.code.sf.net/p/sosie/code/trunk sosie
  cd sosie

  FIX (copied from jdha instead): cp $WDIR/INPUTS/make.macro ./

  make
  make install
  export PATH=~/local/bin:$PATH
  cd $WDIR/INPUTS
  sosie.x -f initcd_votemper.namelist
  sosie.x -f initcd_vosaline.namelist
  $TDIR/WEIGHTS/scripgrid.exe namelist_reshape_bilin_initcd_votemper
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bilin_initcd_votemper
  $TDIR/WEIGHTS/scripinterp.exe namelist_reshape_bilin_initcd_votemper
  $TDIR/WEIGHTS/scripinterp.exe namelist_reshape_bilin_initcd_vosaline

Step 7, weight files for atmospheric forcing::

  $TDIR/WEIGHTS/scripgrid.exe namelist_reshape_bilin_atmos
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bilin_atmos
  $TDIR/WEIGHTS/scripshape.exe namelist_reshape_bilin_atmos
  $TDIR/WEIGHTS/scrip.exe namelist_reshape_bicubic_atmos
  $TDIR/WEIGHTS/scripshape.exe namelist_reshape_bicubic_atmos

Step 8 (mesh and mask for open boundary condition generation)::

  cd $CDIR
  cp $WDIR/INPUTS/cpp_LH_REEF.fcm LH_REEF/
  ln -s $WDIR/INPUTS/bathy_meter.nc $CDIR/LH_REEF/EXP00/bathy_meter.nc
  ln -s $WDIR/INPUTS/coordinates.nc $CDIR/LH_REEF/EXP00/coordinates.nc
  cp $WDIR/INPUTS/runscript $CDIR/LH_REEF/EXP00
  cp $WDIR/INPUTS/namelist_cfg $CDIR/LH_REEF/EXP00/namelist_cfg
  cp $WDIR/INPUTS/namelist_ref $CDIR/LH_REEF/EXP00/namelist_ref
  ./makenemo clean
  ./makenemo -n LH_REEF -m XC_ARCHER_INTEL -j 10
  cd LH_REEF/EXP00
  ln -s $WDIR/xios-1.0/bin/xios_server.exe xios_server.exe

*(16 Jan 2017)* Edit the runscript to include modules and the Account name (n01-NOCL)::

  vi runscript

  #!/bin/bash
  #PBS -N LH_REEF
  #PBS -l select=5
  #PBS -l walltime=00:20:00
  #PBS -A n01-NOCL

  module swap PrgEnv-cray PrgEnv-intel
  module load cray-netcdf-hdf5parallel
  module load cray-hdf5-parallel
  ...

Submit::

  qsub -q short runscript

Doesn't work. No output. I've also tried a fresh rebuild of everything::

  execve error: No such file or directory
  aprun: Apid 24880812: Commands are not supported in MPMD mode
  aprun: Apid 24880812: Exiting due to errors. Application aborted

It looks, to me, like the runscript is missing some flags for something similar.

Just incase tried reloading modules and resubmitting (to standard queue after 8pm)::

  module swap PrgEnv-cray PrgEnv-intel
  module load cray-netcdf-hdf5parallel
  module load cray-hdf5-parallel

  qsub runscript
  4195460.sdb

Still no joy :-(  


---

`Next steps: install pyNEMO <install_pynemo.html>`_