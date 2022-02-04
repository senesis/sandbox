Compatibility with CliMAF of the IS-ENES3 standard interface for pluging diagnostic scripts in evaluation tools

# CliMAF and its scripts interface

## CliMAF basics

`CliMAF <https://climaf.readthedocs.io>`_ is a climate models
evaluation framework developped and used by CNRM, ISPL and CERFACS. It
is Open source and `available on GitHub
<https://github.com/rigoudyg/climaf>`_

CliMAF is basically a Python-scriptable way to process NetCDF `CF
compliant <http://cfconventions.org/>`_ climate model outputs
described through an abstraction : input data are defined by facets
which translate to actual data files through configuration of
so-called data projects.

CliMAF allows :

- to apply diagnostic 'scripts', coded in any language, provided they
  meet very minimal requirements, and they accept command-line
  arguments
- to easily describe piping and combining such scripts, using the
  full flexibility of python
- to keep track of such combinations by building formal expressions in
  a simple syntax (called **CRS** for CliMAF Reference Syntax)
- to trigger CRS expression computation only once needed (this is also
  called 'lazy evaluation')
- to handle a cache of results, which access keys are CRS expressions.

  
## Scripts interface principles in CliMAF 

- the script's data formats are  :

     - NetCDF files for data input
     - NetCDF or graphic files for data output

- NetCDF files must be `CF-compliant <http://cfconventions.org/>`_

- all script parameters are provided as arguments on the script
  command-line; this applies to input and output filenames and to all
  other types of parameters; this implies that other parameters have a
  simple type : number or string; this also implies that scripts do
  not choose the output filenames (CliMAF provides them with filenames
  in its cache);

- CliMAF can pre-process input data regarding selection on variable,
  time and/or space, re-scaling and renaming variables, and
  aggregating data files ; but it can also let the script do it by
  itself for efficiency purpose (thus avoiding creating intermediate
  files)

- the script command line is declared to CLiMAF using patterns for
  arguments, and the syntax of patterns supports their semantics which
  translates script capabilities, among:
  
    - for each input dataset argument : can it be a series of files
      splitted on the time coordinate ? does it represent an ensemble
      ? can the script apply variable renaming, rescaling, or override
      units, or transform a constant to the missing value ? which
      should be the rank of this input in the calling sequence ?
      
    - for each output data : which is its symbolic name (in order to
      distinguish among multiple output types) ? how can CliMAF derive
      the output variable name from the input variable name ?
      
    - for an input dataset representing an ensemble : which are the
      symbolic names for members ?

    - for other parameters : which is there name ?

- this declaration translates to the creation of a python function
  which can be used for scripting operations using CliMAF, which
  basically create CliMAF objects representing the chain of operations
  and their arguments; at a later stage, under user control, such
  objects are 'evaluated', which means that each script involved in
  each sub-object is launched in turn (except if its resultes are
  available in the cache), and its results are stored in the cache

- in the scripting, the function representing the script is used with
  positional arguments representing the input datasets and keyword
  arguments representing the other script parameters (the keywords
  being derived from the patterns used in the declaration); all script
  parameters must be specified on each call (there is no handling of
  default values)

- when the user invokes a script which does not accept ensembles as
  input, and provides it with an ensemble object, CliMAF automatically
  loops invoking the script on each ensemble memeber, and creates as
  call result an ensemble object out of the results set.

- a diagnostic script does not mix computation and graphics, because
  piping both parts (compute and graphics) is easy in CliMAF, and
  second part is prone to be iterated much often than first one (which
  results will then be cached).

Further details are provided in `CliMAF documentation's relevant
section<https://climaf.readthedocs.io/en/master/operators.html#operators>`_

  
## Basic example for interfacing a script with CliMAF

-   Declare operator ``my_cdo`` based on an off-the-shelf
    script/binary (``cdo``)::

     >>> cscript('mycdo','cdo ${operator} ${in} ${out}')

-   Use the defined operator in CliMAF : define a dataset ``tas_ds``
    and apply ``my_cdo`` on it, providing it with value ``tim_avg`` for
    argument ``operator``::

     >>> tas_ds = ds(project='example', simulation='AMIP', variable='tas', period='1980-1981')
     >>> tas_avg = mycdo(tas_ds,operator='timavg')

-   The script/binary is actually called e.g. when requesting a file with
    the content of object ``tas_avg``, as in::

     >>> filen = cfile(tas_avg)

    which returns the filename::

    /home/my/tmp/climaf_cache/4e/4.nc

    ..while the actual system call launched behind the curtain by CliMAF would look like::

     $ cdo tim_avg /home/my/data/AMIP/AMIP_tas.nc /home/my/tmp/climaf_cache/4e/4.nc



# Keypoints of the IS-ENES3 standard interface

The IS-ENES3 standard interface for pluging diagnostic scripts in
evaluation tools (hereafter ISI) is based on a (recommended) script
description file, a master interface file and data definition
files. It assumes that the script is implemented as a command ine tool
uwhich reads the master interface file.

The script definition file provides the list of mandatory and optional
keys for the script, and further references (to litterature and
documentation); it does not describe in a structured way the list of
outputs generated by the script.

The master interface file provides the name of the data definition
files, the values of all script input parameters, and ancilliary
information (such as a log level and imposed working directories)

Each data definition file contains for a series of input file : the
name of the variable, an `alias`, and the filename for all input data
for this alias+variable; it may contain metadata (based on the CMIP6
Data Reference Syntax) for complementing data description; it may also
contain a start and end date for the data, and the `alias` for a
reference dataset that could apply to the data. The semantics for
start and en date is not specified (should it match the data content,
or should the script use it for selecting a period in the datafile
?). Similarly, the intent for providing metadata that are redundant
(and possibly inconsistent) with NetCDF metadata in files is not
explicit.


# Compatibility between the IS-ENES3 standard interface and CliMAF

##  Shallow compatibility
(implémentation dans CLiMAF)

## Deep compatibility
   ? pre-processing

## CliMAF changes needed for compatibility

## Changes to the standard needed for compatibility
