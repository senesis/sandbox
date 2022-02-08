# Compatibility with CliMAF of the IS-ENES3 standard interface for pluging diagnostic scripts in evaluation tools

Stéphane Sénési and Jérôme Servonnat, IPSL, 

February 2022

## Abstract
CliMAF is largely compatible with the IS-ENES3 standard interface, and
this was demonstrated by an actual implementation using the parent
interface (ESMValTool's).  It is thus possible to execute a script
that uses the standard interface from CliMAF.  However, CLiMAF looses
its smart cache functionnality when executing a script using the
standard interface: it cannot handle script outputs in its cache
because the standard interface does not describe script outputs before
script execution.

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Compatibility with CliMAF of the IS-ENES3 standard interface for pluging diagnostic scripts in evaluation tools](#compatibility-with-climaf-of-the-is-enes3-standard-interface-for-pluging-diagnostic-scripts-in-evaluation-tools)
    - [Abstract](#abstract)
    - [CliMAF and its scripts interface](#climaf-and-its-scripts-interface)
        - [CliMAF basics](#climaf-basics)
        - [Scripts interface principles in CliMAF](#scripts-interface-principles-in-climaf)
            - [Data formats](#data-formats)
            - [Use command line](#use-command-line)
            - [Propose basic pre-processing](#propose-basic-pre-processing)
            - [Declare script command line](#declare-script-command-line)
            - [... which creates a python function (or `operator`)](#-which-creates-a-python-function-or-operator)
            - [Match `operator` call sequence to script parameters](#match-operator-call-sequence-to-script-parameters)
            - [Automate processing ensembles](#automate-processing-ensembles)
            - [Separate graphics and numeric outputs](#separate-graphics-and-numeric-outputs)
        - [Basic example for interfacing a script with CliMAF](#basic-example-for-interfacing-a-script-with-climaf)
    - [Keypoints of the IS-ENES3 standard interface](#keypoints-of-the-is-enes3-standard-interface)
    - [Compatibility between the IS-ENES3 standard interface and CliMAF](#compatibility-between-the-is-enes3-standard-interface-and-climaf)
        - [Shallow compatibility](#shallow-compatibility)
        - [Deep compatibility](#deep-compatibility)
        - [CliMAF changes needed for compatibility](#climaf-changes-needed-for-compatibility)
        - [Changes to the standard needed for compatibility](#changes-to-the-standard-needed-for-compatibility)
        - [Compatibility proof-of-concept](#compatibility-proof-of-concept)

<!-- markdown-toc end -->


## CliMAF and its scripts interface
### CliMAF basics

[CliMAF](https://climaf.readthedocs.io) is a climate models
evaluation framework developped and used by CNRM, ISPL and CERFACS. It
is Open source and [available on GitHub](https://github.com/rigoudyg/climaf)

CliMAF is basically a python-scriptable way to process NetCDF climate
model outputs described through an abstraction : input data are
defined by facets which translate to actual data files through
configuration of so-called data projects.

CliMAF allows to :

- apply diagnostic 'scripts', coded in any language, provided they
  meet very minimal requirements, and they accept command-line
  arguments
- easily describe piping and combining such scripts, using the
  full flexibility of python
- keep track of such combinations by building formal expressions in a
  simple syntax (called **CRS** for CliMAF Reference Syntax); the CRS
  is notably a way to document the provenance of all CliMAF result and
  to build a unique identifier for each result
- trigger CRS expression computation only once needed (this is also
  called 'lazy evaluation')
- handle a cache of results, which access keys are CRS expressions.

### Scripts interface principles in CliMAF 
#### Data formats
The script's data formats are  :

  - NetCDF files for data input
  - NetCDF or graphic files for data output

and NetCDF files must be [CF-compliant](http://cfconventions.org/)

#### Use command line
  All script parameters are provided as arguments on the script
  command-line; this applies to input and output filenames and to all
  other types of parameters; this implies that scripts do
  not choose the output filenames (CliMAF provides them with filenames
  in its cache). Under these assumptions, **CliMAF requires no change
  to pre-existing scripts**.

#### Propose basic pre-processing
  CliMAF can pre-process input data regarding selection on variable,
  time and/or space, re-scaling and renaming variables, and
  aggregating data files ; but it can also let the script do it by
  itself for efficiency purpose (thus avoiding to create intermediate
  files)

#### Declare script command line
The script command line is declared to CLiMAF using patterns for
arguments, and the pattern syntax supports their semantics, which
translates script capabilities, among:  

- for each input dataset argument : can it be a series of files
  splitted on the time coordinate ? does it represent an ensemble
  ? can the script apply variable renaming, rescaling, or override
  units, or transform a constant to the missing value ? which
  should be the rank of this input in the calling sequence ?
- for each output data : which is its symbolic name (in order to
  distinguish among multiple outputs) ? how can CliMAF derive
  the output variable name from the input variable name ?
- for other parameters : which is their name ?
- can the script select data in datafiles based on the variable
  name, the time period, the space domain ?
 - for an input dataset representing an ensemble : which script
  argument should receive the symbolic names for members, if
  needed ?

#### ... which creates a python function (or `operator`)
  This declaration translates
  to the creation of a python function (called a `CliMAF operator`)
  which can be used for scripting operations using CliMAF, which
  basically create CliMAF objects representing the chain of operations
  and their arguments; at a later stage, under user control, such
  objects are 'evaluated', which means that each script involved in
  each sub-object is launched in turn (except if its results are
  available in the cache), and its results are stored in the cache

#### Match `operator` call sequence to script parameters
  In the scripting, the function representing the script is used with
  positional arguments representing the input datasets and keyword
  arguments representing the other script parameters (the keywords
  being derived from the patterns used in the declaration); all script
  parameters must be specified on each call (there is no handling of
  default values)

#### Automate processing ensembles
  When the user invokes a script which does not accept ensembles as
  input, and provides it with an ensemble object, CliMAF automatically
  loops invoking the script on each ensemble memeber, and creates as
  call result an ensemble object out of the results set.

#### Separate graphics and numeric outputs
  A diagnostic script does not mix computation and graphics, because
  piping both parts (compute and graphics) is easy in CliMAF, and
  second part is prone to be iterated much often than first one (which
  results will then be cached).

Further details are provided in [CliMAF documentation's relevant
section](https://climaf.readthedocs.io/en/master/operators.html#operators)

  
### Basic example for interfacing a script with CliMAF
Declare operator ``my_cdo`` based on an off-the-shelf
script/binary (``cdo``):

       cscript('my_cdo','cdo ${operator} ${in} ${out}')

Use the defined operator in CliMAF : define a dataset ``tas_ds``
and apply ``my_cdo`` on it, providing it with value ``timavg`` for
argument ``operator``:

       tas_ds = ds(project='example', simulation='AMIP', variable='tas', period='1980-1981')
       tas_avg = my_cdo(tas_ds,operator='timavg')

The script/binary is actually called e.g. when requesting a file with
the content of object ``tas_avg``, as in:

       filen = cfile(tas_avg)

cfile both executes the script involved in ``tas_avg`` (if result is
not yet available in cache) and returns the filename (built by CliMAF
using the user provided path to the CliMAF cache and a hash of the
CRS):

      /home/my/tmp/climaf_cache/4e/4.nc

..while the actual system call launched behind the curtain by CliMAF would look like:

       $ cdo timavg /home/my/data/AMIP/AMIP_tas.nc /home/my/tmp/climaf_cache/4e/4.nc

## Keypoints of the IS-ENES3 standard interface

The IS-ENES3 standard interface (hereafter ISI) for pluging diagnostic
scripts in evaluation tools (hereafter `tools`) is described in the main
body of the present document.

It is based on a (recommended) script
description file, a master interface file and data definition
files. It assumes that the script is implemented as a command line
tool which is driven by the master interface file content. The
interface files are in Yaml format.

The optional, static **script definition file** provides the list of
mandatory and optional keys (i.e. parameter names) for the script, and
further references (to litterature and documentation); it does not
describe in a structured way the list of outputs generated by the
script.

The dynamic **master interface file** provides the name of the data definition
files, the user-provided values of script input parameters, and ancilliary
information (such as a log level and imposed working directories)

Each dynamic **data definition file** contains for a series of input
files : the name of the variable, an `alias`, and the filename for all
input data for this alias+variable; it may contain metadata (based on
the CMIP6 Data Reference Syntax) for complementing data description;
it may also contain a start and end date for the data, and the `alias`
for a reference dataset that could apply to the data. The semantics
for start and enddate is not specified (should it match the data
content, or should the script use it for selecting a period in the
datafile ?). Similarly, the intent for providing metadata that are
redundant (and possibly inconsistent) with NetCDF metadata in files is
not explicit.


## Compatibility between the IS-ENES3 standard interface and CliMAF
CliMAF is a natural candidate for being one of the `tools` using the ISI.
We will comment on two levels of compatibilities:
- shallow compatibility: can CliMAF execute a script that follows the ISI?
- deep compatibility: does the ISI affect some functionalities of CliMAF?

###  Shallow compatibility
CliMAF interface design is quite different from ISI's:

  - it does not assume that the script is changed to adapt to the
    interface, and it adapts to the existing script command line
    structure;
  - it communicates all script parameters and input data references
    through the command line, while ISI uses the master interface
    file;
  - it treats separately data ensembles and single members, while ISI
    handle single members as an ensembles of size one;
  - part of the information provided in ISI's script definition file
    is in CliMAF supported by [the script declaration
    phase](#declare-script-command-line)

Nevertheless, regarding feeding the script with input data and
parameters, there is no basic incompatibility, as both interfaces have
necessarily to provide the same basic set of information.

Three other differences can be noted :

- ISI does not include any feature allowing to avoid pre-processing
  data before calling the script (see the [script declaration
  phase](#declare-script-command-line). While this is not a real
  incompatibility, this means that for a number of use cases,
  un-necessary intermediate files may have to be generated when using
  ISI.

- ISI allows for parameter values which have complex types (those
  allowed by yaml); this is inherited from ESMValTool, with which the
  user can also use yaml, in recipes, for providing parameter values;
  CLiMAF has to code parameter values in strings (for the command
  line)

- CliMAF allows only for NetCDF and graphics outputs, and does
  not allow having both types for a single script, while ISI does not
  impose any constraint about outputs.


### Deep compatibility
The most significant difference between ISI and CLiMAF interface is
about declaring script's outputs : ISI provides no way to declare how
many outputs the script will generate, nor to label them, while CliMAF
needs such a declaration ahead of script execution. CLiMAF uses the
output labels for naming the actual outputs by combining it with the
the CRS expresssion representing the script call. This is instrumental
in CLiMAF's logic for caching all results, including in order to
re-use outputs in further processing.  Therefore, all CliMAF
functionalities downstream of the execution of a script (cache, use of
CliMAF plotting scripts, build of an html page using the CliMAF html
toolbox) are not directly usable.

### CliMAF changes needed for compatibility
The main changes that would be needed in CliMAF for interfacing with
scripts matching the ISI are :

- to create a function allowing both to declare an ISI compatible
  script and to create a corresponding python function (called an
  `ISI-operator`) usable in CliMAF scripting

- to create a driver function able to :

  - build ISI compliant master interface file and data definition
    files, based on datatets and keyword parameters used in calls to
    an `ISI-operator`
  - create user-required working directories
  - call the script (taking care if needed of setting environment) 
  - return basic information on script success and results location

Regarding the question of converting to ISI those scripts that are
already interfaced with CliMAF : it is not deemed practicable for most
of these scripts which are actually binaries that live outside CliMAF
(such as CDO commands) and hence are not easy to modify; for the few
other ones, the change is not necessarily useful

### Changes to the standard needed for deep compatibility
The sole change in ISI that is instrumental for CLiMAF to take full
advantage of ISI compatible scripts in CliMAF is related to the deep
compatibility issue invoked [above](#deep-comatibility). It would be
to make the (recommended) script definition file mandatory and to
include a manadatory script outputs declaration section; such a
section would include, for each output :

- a label allowing CliMAF to name this output
- a pattern allowing CliMAF to find this output in one of the working
  directories

CliMAF would then require further changes for reading the definition
file, and for searching for script outputs and storing them in cache.
A further change would be to handle more varied type of outputs (and
not only graphics and NetCDF files)

### Compatibility proof-of-concept 
In order to check [the analysis
above](#climaf-changes-needed-for-compatibility) about CliMAF changes
needed for ISI compatibility, we took advantage of the fact that ISI
inherits from [ESMValTool scripts
interface](https://docs.esmvaltool.org/projects/esmvalcore/en/latest/interfaces.html),
and implemented in version 2.0.2 of CliMAF the changes needed for
interfacing with version 2.3 of ESMVaTool, as [described in CLiMAF
documentation](https://climaf.readthedocs.io/en/master/esmvaltool.html).
This was successfully tested for calling the Climate Variability
Diagnostic Package (CVDP) embarked in ESVMavTool, and the code below 
shows the corresponding CliMAF script.


      # An example of declaring and calling an ESMValTool script from CliMAF
      
      from climaf.api import *
      from climaf.ESMValTool_diags import evt_script
      
      # If your platform is not Ciclad, you must tell which is the wrapper for ESMValTool scripts
      climaf.ESMValTool_diags.wrapper = "~/scripts/ESMValTool_python_diags_wrapper_for_ciclad.sh"
      
      # Create a CliMAF function for calling the ESMValTool diagnostic script
      # (use the same syntax as the ESMVaTool recipe for designating the script)
      evt_script("call_cvdp", "cvdp/cvdp_wrapper")
      
      # Prepare input datasets for the diag. 
      base      = dict(project="CMIP6", experiment="historical",
                       realization='r1i1p1f2',  table="Amon", period="1850-1855", )
      models    = [ "CNRM-CM6-1", "CNRM-ESM2-1"]
      
      variables = [ "ts", "tas", "pr", "psl" ]
      
      ensembles = []
      for variable in variables:
          ensemble = cens(
              {
                  model :  ds(model=model, variable=variable, **base)
                  for model in models
              })
          ensembles.append(ensemble)
      
      # Note : here, for other diagnostic scripts, you may have to reproduce
      # the preprocessing steps that ESMValTool recipes implement upstream
      # of the diagnostic script. For CVDP, there is actually no such
      # preprocessing
          
      # Call the diag. You may provide parameters that are known to ESMValTool
      # or to the diagnostic script
      wdir, prov = call_cvdp(*ensembles, output_dir="./out", write_netcdf=False)
      
      # First returned value is the diag's working directory
      print(wdir)
      
      # Second one is a dictionnary of provenance information which
      # describes all outputs (either graphics or NetCDF files) by various
      # attributes, one of which being a 'caption'
      one_output, its_attributes=prov.popitem()
      print(one_output, its_attributes['caption'])
