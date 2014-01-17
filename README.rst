=================================
Tailor your SteamOS to your needs
=================================

Motivation
----------
SteamOS is built on top of Debian - using a generic kernel which is configured
to run out-of-the-box on as many hardware configurations as possible.
The default `.config` file contains almost 4200 compiled features in both the main
binary and about 3000 additional loadable modules (such as drivers).
Generally it's okay to have a Fiber Distributed Data Interface [#fddi]_ or Braille
[#braille]_ support in the Linux Kernel, but do you really need it for your
SteamOS?
Although it sometimes occurs that removing a feature will slightly speedup your
system, this isn't the main goal. As shown before [#ndss13]_, this approach
might help you to avoid (yet undiscovered) bugs or their exploitation.


Introduction
------------
We are working on a university project focusing on variability in operating
systems (mainly in the Linux kernel).
Some parts of the tools have the ability of building a kernel configuration out
of a number of source code points.
In short the approach is performed in the following four steps:

1. Recording executed kernel addresses while running your system in its
   default use case (using the built in function tracer [#ftrace]_)
2. Translating them to code points in the kernel source
   (via binary debug information [#dwarf]_)
3. Generating a logical expression out of their variability requirements
   (analyzing the block and file preconditions)
4. Solving the expression (using an external tool [#picosat]_ for the
   satisfiability problem)

To adapt the tools to SteamOS we did some (small) changes to the original tools
published by the VAMOS project [#vamos]_.

Although this approach could generate a perfect working kernel without any
additional work, this is primarily supposed to be a helper tool to
detect important features.


Howto
-----
0.  Since SteamOS doesn't provide all packages for compiling a Linux kernel,
    the easiest way is to work on an additional (x86_64) system with Debian or
    Ubuntu.

1.  Get the sources from the SteamOS repository::

        wget http://repo.steampowered.com/steamos/pool/main/l/linux/linux-source-3.10_3.10.11-1+steamos8+bsos1_all.deb
        dpkg -x linux-source-3.10_3.10.11-1+steamos8+bsos1_all.deb .
        tar -Jxf usr/src/linux-source-3.10.tar.xz

2.  Build a traceable kernel with debug symbols::

        cd linux-source-3.10/
        xzcat ../usr/src/linux-config-3.10/config.amd64_none_amd64.xz > .config
        echo -e "#Required for tracing
        CONFIG_KPROBES_ON_FTRACE=y
        # CONFIG_PSTORE_FTRACE is not set
        CONFIG_DEBUG_INFO_REDUCED=y
        CONFIG_FUNCTION_TRACER=y
        CONFIG_FUNCTION_GRAPH_TRACER=y
        CONFIG_DYNAMIC_FTRACE=y
        CONFIG_DYNAMIC_FTRACE_WITH_REGS=y
        # CONFIG_FUNCTION_PROFILER is not set
        CONFIG_FTRACE_MCOUNT_RECORD=y" |sed -e "s/^ *//" >> .config
        make deb-pkg -j$(grep -c ^processor /proc/cpuinfo)
        cd ..

    You should get three Debian packages named
        - linux-image-3.10.11_3.10.11-1_amd64.deb
        - linux-headers-3.10.11_3.10.11-1_amd64.deb
        - linux-libc-dev_3.10.11-1_amd64.deb

    Next you should build the `undertaker-traceutil` from the Github repository
    [#github_] (slightly modified compared to the original research version
    for the use with Debian based systems) by running `make` in this directory::

        git clone https://github.com/i4vamos/steamos.git
        cd steamos
        make
        cd ..

    Transfer the whole *steamos* directory along whith the three Debian packages
    to the target (steam) machine.

3.  Download the *Undertaker* tool and prepare the Linux kernel sources for the
    following steps.

    The *Undertaker* tool has a few dependencies to build (e.g. libboost).
    If compilation fails on your machine and it's not obvious which package
    might be missing, take a look at the project site [#vamos]_ where we also
    provide a copy-and-paste-able install command for all dependencies.
    If there's still a problem, don't hesitate to contact us.

    To download and compile the tool, enter the following commands::

        git clone https://vamos.informatik.uni-erlangen.de/gerrit/p/undertaker.git
        cd undertaker
        make -j$(grep -c ^processor /proc/cpuinfo)
        cd ..

    Next, you need to prepare the previously downloaded Linux kernel source
    code. Our tools build a model which comprises information about the
    different dependencies between selectable features.
    To build this model, type the following commands::

        PATH=$PATH:$(pwd)/undertaker/scripts/kconfig/:$(pwd)/undertaker/python/:$(pwd)/undertaker/undertaker/
        export PATH
        cd linux-source-3.10/
        cat ../undertaker/python/undertaker-kconfigdump | grep -v "^PATH=" > undertaker-kconfigdump
        chmod +x undertaker-kconfigdump
        ./undertaker-kconfigdump -i x86

    This might take a while to complete (up to a few hours).

4.  Meanwhile you can prepare the target steam machine.
    Since there is no *ssh-server* installed (by default), you may use a text
    terminal (e.g. *Ctrl*+*Alt*+*F3*).
    First you have to install the new kernel by running ::

        sudo dpkg -i linux-*.deb

    Reboot the system and run the following command from inside the
    copied *steamos* directory::

        sudo ./undertaker-tracecontrol-prepare

    If you have no *grub* boot menu, you can change the names of your initial
    ramdisk to swap them, for example::

        sudo mv initrd.img-3.10.11 initrd.img-3.10.11.org
        sudo mv initrd.img-3.10.11.ftrace initrd.img-3.10.11

    Otherwise you need to edit the boot parameter. In *grub* select the new
    kernel and press the *e* key. Change the `initrd` entry to the generated
    ramdisk (by appending `.ftrace`)::

        initrd /boot/initrd.img-3.10.11.ftrace

    and continue bootup by pressing the `F10` key.

5.  The bootup may take a bit longer than usual: this is due to the system
    collecting addresses. The file `/run/undertaker-trace.out` should contain
    a few thousand lines with hexadecimal values (representing the called
    addresses).

    Use your system as you would typically use it. The trace tool will record
    which functions have been called inside the kernel and log these addresses.

    **IMPORTANT:** At absolutely **no** point in time do we have access to
    **any personal data** inside the kernel - it's only about addresses in the
    code!

    After a sufficient time (something between 10 minutes and an hour) save a
    copy of `/run/undertaker-trace.out` and transfer it back to your build
    machine into the top level folder.

6.  Once these steps have been completed, you can actually start to generate a
    kernel!
    Make sure that the *steamos/lists* directory is available on the system.
    Enter the following commands to start the analysis (using the binaries
    generated in step 2)::

        cd linux-source-3.10/
        ../undertaker/tailor/undertaker-tailor \
            -b ../steamos/lists/blacklist.steam \
            -w ../steamos/lists/whitelist.steam \
            -i ../steamos/lists/undertaker.ignore \
            -m models/x86.model -u ../undertaker/undertaker/undertaker -s . \
            -k debian/ -e vmlinux ../undertaker-trace.out > trace.config

    Since the non-ternary config items (mostly numbers and strings) cannot be
    guessed well, they must be extracted from the original config::

        cat .config | grep -v "^#\|=y$\|=m" | sort -u >> tailor.config

    Expand and build the kernel based on the new configuration ::

        make KCONFIG_ALLCONFIG="tailor.config" allnoconfig
        make deb-pkg -j$(grep -c ^processor /proc/cpuinfo)

    Transfer the new debian packages to your steam machine and install them
    using ::

        sudo dpkg -i linux-*.deb

    and reboot. The system *should* boot without errors in your new system.
    But if *not* - you need to compare the original configuration with the
    generated one and find the missing or spurious features.
    You can add them to the white- or blacklist for default in- or exclusion.

7.  Have fun!


Comments
--------
During our adaption of the tools for SteamOS, we generated a config on a system
with a Core i7 processor and a Nvidia Titan graphics card.
The result can be found at [#config_paste]_. The number of features was reduced
from **4191** to only **616**, with everything still working fine.


Limitations
-----------
- Depending on your system it could happen that the tools aren't able to
  generate a solution. This is due to technical issues: the model doesn't have
  a 100% accuracy (but almost!) and under some special circumstances it won't
  get it right.
- Some necessary features might be missing because of untraceable functions (or
  perhaps they aren't even generating traceable code). You can add them using
  the whitelist. To recognize such features it might be helpful to take a look
  into the original configuration. Take special attention towards features
  involved in the early boot process.
- Sadly, it cannot do magic. If your trace run didn't contain your complete
  usecase, some features **might** be missing. Especially different hardware
  components should be tested.


License
-------
See `LICENSE` for the **GNU GENERAL PUBLIC LICENSE**


References
----------

.. [#fddi] Default kernel config: `CONFIG_FDDI=y`
.. [#braille] Default kernel config: `CONFIG_A11Y_BRAILLE_CONSOLE=y`
.. [#ndss13] http://www4.cs.fau.de/Publications/2013/kurmus_13_ndss.pdf
.. [#ftrace] https://www.kernel.org/doc/Documentation/trace/ftrace.txt
.. [#dwarf] http://dwarfstd.org/
.. [#picosat] http://fmv.jku.at/picosat/
.. [#vamos] http://vamos.informatik.uni-erlangen.de/trac/undertaker
.. [#github] http://github.com/i4vamos/steamos
.. [#config_paste] http://pastebin.com/23r64hYq
