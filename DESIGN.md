# QInstall design documentation

qinstall performs all of its actions by acting on a live directory
structure, so the layout is critical for it's operation. Everything is
dynamically read from the file system and there is no separate cache. This
makes it easy to start using the qinstall system on a new machine or
platform.

The `QROOT` variable is critical to qinstall's operation. Assuming you do
not specify a `QROOT` variable in the environment provided to qinstall, the
tool will search for a qinstall root in the following locations in
sequence:

* `/qinstall` (The most common place to mount the qinstall root)
* The directory where the qinstall utility was executed from (This was
     not chosen as a default in case you choose you symlink or copy the
     qinstall tool into an alternate directory in your path, such as
     /usr/local/bin)

qinstall then checks if any of the generic parameters were specified on
the command line and processes them:

* **-d _directory_** Sets the QINSTALLDIR parameter to the specified directory. This causes qinstall to search for the image to install from the specified directory
* **-p** Sets the `QPREREQS` variable to '1' which triggers the installation of all prerequisite products where possible.
* **-R _responsefile_** Sets the QRESPFILE variable which is frequently used by the .qinstall scripts to override the default response files for the product install

The `QPLAT` variable is then set to the correct value for the platform we
are running on. This is usually from the uname variable, but there are some
special cases for AIX and HP-UX. The product and version parameters are set
into the `QPRODUCT` and `QVERSION` variables.

If a product name, or version name is missing, qinstall will then display
the list of available options to the user

qinstall sets `QLOCATION` to the directory which contains the .qinstall
scripts, and `QINSTALLDIR` is set to the same location if it hasn't already
been set (either in the environment or via the -d parameter described
earlier)

If the prereqs option has been set, then qinstall finds any suitable
`.qprereqs` file, and recursively calls qinstall for each prerequisite product
listed in the file, specifying the `-p` option so that prerequisites are
installed recursively.

The `.qinstall` script can be located in a number of different locations,
so a for loop is executed which checks each of the possible locations (from
highest priority to lowest). Once a script is found, the following takes
place, and once a script has been executed, the loop is broken (the EXECUTED
variable is used to check for this condition):

* The location of the selected .qinstall script is displayed for debugging
* The `.qinstall` script is scanned for the `LOGFILES=` definition, and the LOGS variable is set to the list of logs defined.
* The `.qinstall` script is executed and the output logged

Once this is done, an entry is written to either `/var/log/qinstall.log` or
`/var/adm/qinstall.log` to indicate what has been done assuming this can be
written to (since softweare installations are commonly done with
root privileges this shouldn't be a problem
