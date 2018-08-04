# QInstall - fast, repeatable product installs over networked file systems

## Executive Overview

QInstall is a tool initally developed at IBM for getting (mostly proprietory,
commercial) software up and running quickly which can often be problematic
for end users and result in precious time being lost.  It uses a
network-shared file system (such as NFS or even something like sshfs) hosting
 the product installers, the qinstall generic utility, and a set of response
files for silent installs.  It solves the problem of locating, installing
and getting up and running quickly with software, while making the whole
process transparent so that it can be understood and modified by anyone if
required.  In addition to being quick (running the installer from a network
drive avoids disk contention from copying it to the host first) installing
from a consistent repository allows repeatable, portable installation of
environments from scratch to avoid reliance on undocumented environment
changes which are common on many systems.  With
[ULM](https://github.com/IBM/ulm) integration, qinstall can now
automatically monitor the installation logs in real time to show you what's
happening, this providing an advantage over most GUI installers

In addition to software installations, qinstall has been used to "install"
(i.e.  create) virtual machines using various technologies across
architectures including containers on Linux, zones and logical domains on
Solaris, WPARs on AIX, and Integrity VMs on HP-UX via a common interface.

[For design documentation, click here](DESIGN.md)

## OK, so what is it really that you're providing?

Put simply, it's the contents of a network mounted filesystem that you can 
host on your intranet containing installable product drivers, which are
annotated with a standardised script that someone with the knowledge can
produce and share for others to use in a standard manner. Scripts are produced
per-product to installs the product, optionally (but recommended) another to
uninstall, another optional one to perform an initial "sanity test" of the
product (and is simple enough that it can be used to show how to use the
product so is partially educational) plus any appropriate customised response
files to allow the install and uninstall to take place. These customised
scripts for install etc. are stored on the NFS file system along with the
product in a specfic directory structure. Some scripts will be hosted
on the github repo where you get qinstall from, others make come from elsewhere


## What does it buy me?

* Bypassing the GUI with a noninteractive installer makes the installs run quicker (DB2 installs in about a minute on my test systems)
* You can install a product without knowing too much about it
* The installations are consistent and repeatable
* Once you've integrated a product into qinstall, it is usually trivial to port it to other platforms
* install can be configured to install all product prereqs
* A separate, optional, IVT step can check that the product is installed
and working, and can act as an example of how to perform basic operations.
* Transparent operation - you can easily see what qinstall is doing, which
means if you have custom requirements it is easy to change

## Sounds great! How do I use it?

You can set up your own server by extracting from the github project:

```
git clone https://github.com/sxa555/qinstall
```
then extract the product specific scripts for the ones you wish to use - it
doesn't extract all of them by default so as not to fill up your filesystem
with confusing files that won't work. The ones I've written are in branches
of the qinstall project, but these can be taken from anywhere that you store
qinstall scripts e.g.
```
cd qinstall
git clone https://github.com/sxa555/qinstall -b db2
```

Edit the top level qinstall/quninstall/qivt files to point at the place
you've extracted them to if it was somewher other than the default location
`/qinstall`, put the downloaded installers in place, then share it over NFS
or another networked filesystem out to all your client machines - the ones
you want to install products onto.

As an example, if you wanted to install db2, you can query the list of
available versions known to qinstall:

```
# qinstall db2
/qinstall/qinstall[31]: SunOS_i86pc_:  not found
That version is not available. Pick one from this list:
82
82U
82U14
82U14X
91
```
Then install the appropriate version with:

```
# qinstall db2 82
```

Simple! If you want to use the existing qinstall scripts to install a driver
from another location (e.g. a nightly build of your product, or a version that 
isn't currently on your qinstall server, then you can
use the -d flag to tell qinstall where to install from as follows:

```
# qinstall -d /builds/product/x123456 db2 82
```

The directory structure used by qinstall is critical to how it functions and is as follows
(some of these version numbers are a little out of date but are from when I
initially developed it!)

```
/qinstall --+-- qinstall
	   +-- quninstall
	   +-- qivt
	   +-- db2 --+-- Linux_i686_82
	   |         +-- SunOS_sun4u_82
	   |         |-- SunOS_sun4u_82U --+-- 4
	   |         |                     +-- 5
	   |         +-- 91
	   |         +-- SunOS_sun4u_91
	   |         +-- (etc.)
	   +-- was ----- (as for db2)
	   +-- mq  ----  (as for db2)
	   +-- (etc)
```

The above structure shows you where the products are on the system. Versions
with 'U' are fixpack updates, not full installs. The two previous parts of
the directories, separated by underscores, are from the output of the
'uname' command.  The bottom level directories contain:<p>

* The product-specific qinstall scripts
* The product installer itself

Directories containing version numbers without a platform (e.g. db2/91 in
the above diagram) are used to hold default qinstall scripts which are used
on any platforms which don't have their own ones. This allows you to create
qinstall support for all platforms without having to have duplicate scripts.
In general, such directories will not hold product code, only qinstall
control files, although this is not required.

# How do I add a new product?

the best way to get started is to have a look at an existing one as an
example.

You need to supply the following scripts. Not all are mandatory, but it is
recommended to include all of them. They MUST all be non-interactive, but
can take command line parameters. If there are mandatory parameters, then
the script should abort immediately with a suitable usage description rather
than prompt the user:<p>

| SCRIPT | MANDATORY? | FUNCTION |
|---|---|---|
|.qinstall | Yes | Installs the product. It should check that the product is not already installed first before proceeding (usually by checking for the presence of an installed file) and verify afterwards that the install has been successful (same check). It will usually make use of a custom response file. To allow external response files to be specified, you should parse the QRESPFILE variable if it has been set using a construct such as `QRESPFILE=${QRESPFILE:-$QLOCATION/.qdefault.rsp`. The scripts may take parameters to customise the install if required. By adding a `QLOGFILES=` directive to the script which points at the (space separated) full path to all installation logs created during the install,  qinstall will use ULM to display the logs in real time where possible or will show these at the end of the install. Generally the install scripts will not perform any product instance creations (a demo of how to do one can be performed in the `.qivt` script instead) but creation of mandatory users/groups withinin `.qinstall` is encouraged. If you create users/groups, the presence of those already on the system should not cause an error. After completion, use an echo command to tell the user where any relevant install logs are for debugging. NOTE: The script will be invoked from the directory containing the product installer. When qinstall is invoked with the -d flag, this may not be the directory the .qinstall script or response files.  Therefore references to other scripts which are held in the qinstall system, should be made as $QLOCATION/file - the QLOCATION variable will always be set to the location of the scripts. See the logic in the top level qinstall script to see how this is done. |
| .quninstall | Yes | Reverses the above install. The uninstall is not required to remove any specific user data (for example DB2 instances) but this can be done optionally if the script receives a -f flag. After completion, use an echo command to tell the user where any relevant uninstall logs are for debugging |
| .qsource | No | Text file specifying where the installed product came from. Each line should be in the format PLATFORM_VERSION:&nbsp;URL.  The URL can be any normal URL |
| .qproduct.rsp | No | Used if the .qinstall/.quninstall needs it.  The name of this file is flexible, .qSOMETHING.rsp is recommended.  You should always reference response file scripts as $QLOCATION/.qSOMETHING.rsp - never with an absolute or relative path.
| .qivt | No | This script is used to verify that the installed worked. It should run through a series of operations on the product, and also serves as a script people can look through to show how to use the product without reading through all of the documentation.  It is recommended that any objects created by this script are called "q_ivt", and any objects created should be removed afterwards.  <tr><td>.qprereqs<td>No<td>A list of prerequisite products that need to be installed before this product will work. If the top level qinstall script is invoked with the '-p' parameter then this file will be processed, and all prereqs listed will be qinstalled before this product is installed. The file contains one line per product, each line is the parameters that need to be passed to qinstall to install the product (e.g. 'was 61'). The file will be parsed in order, and each implicit call to qinstall will recieve the '-p' parameter to perform recursive prereq installation. No loop checking is done :)

All of these scripts are prefixed with '.' so they do not normally display
in a directory listing. Therefore each product directory looks like the
normal product installer. The product's own files should never be modified.
Any sample response files should be copied (preferably into a name as
described in the table above) before editing, and put into the repository
along with the scripts.

Scripts should return zero on success, or if the product is already
installed/uninstalled, non-zero if the install failed.

The following flags are reserved for the qinstall system itself and should
not be used by your own scripts:

* **-d _DIRECTORY_ ** Install the product from a different location not
     provided by qinstall e.g. a directory containing a development drivers
* **-p** Install all pre-requesite products first. The list of products to
     install comes from the .qprereqs file in the product directory.
* **-R _RESPONSEFILE_ ** Specify an alternate response file to be used
     during the install. Most likely used in the .qprereqs file for products
     which need to customize a prerequisite product .
* **-k** Reserved - not yet implemented - Update kernel parameters on the OS
* **-f** Reserved - not always implemented - Perform a full uninstall
     including all user IDs etc.

Once you've written the scripts for your new product, you should test them
(ideally on a clean OS install) by making sure the following sequence of
operations works correctly:

* qinstall
* quninstall
* qinstall *(with -d to specify an install image in an alternate location)*
* qivt
* quninstall

### Tips/snippets for writing your own qinstall scripts in ksh

As mentioned above, the qinstall should ideally check for the presence of the
product on the system before trying to install. With a ksh script, this is
the easist way to check and fail if it is already there:

```
   [ -r /opt/IBM/db2/V9.1/instance/db2icrt ] && echo Already installed && exit 0
```

After install, check if it has worked properly;

```
   [ ! -r /opt/IBM/db2/V9.1/instance/db2icrt ] && echo Install failed && exit 2 || exit 0
```

The `|| exit 0` at the end should only be used if this is the last line of
the script. Without that, the script will give a non-zero exit code - the
result of the `! -r` test.<p>

Here is a ksh snippet that shows how to parse two parameters in a .qinstall
script. The first is '-a' which takes a value, and the second is '-s',
which doesn't. The ':' in the getopts line indicates that the previous
option takes a parameter, and it is referenced as $OPTARG within the case
statement:

```
   while getopts "a:s" opt; do
        case "$opt" in
                "s") RESPONSEFILE=.qsecure.rsp
                     ;;
                "a") ENABLEALTERNATIVE=$OPTARG
                     ;;
        esac
   done
```

## Future features

Raise a github issue to tell me what you'd like to see or if you have a
problem or question, but here are some that I've prototyped in one form or
another ...

* Automatic product downloads
* Live monitoring of the installation logs
* Improved Windows support

