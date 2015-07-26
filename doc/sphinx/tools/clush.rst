.. _clush-tool:

clush
-------

.. highlight:: console

*clush* is a program for executing commands in parallel on a cluster and for
gathering their results. It can execute commands interactively or can be used
within shell scripts and other applications. It is a partial front-end to the
:class:`.Task` class of the ClusterShell library (cf. :ref:`class-Task`).
*clush* currently makes use of the Ssh worker of ClusterShell that only
requires *ssh(1)* (we tested with OpenSSH SSH client).

Some features of *clush* command line tool are:

* two modes of parallel cluster commands execution:

  + **flat mode**: sliding window of local or *ssh(1)* commands
  + **tree mode**: commands propagated to the targets through a tree of
    pre-configured gateways; gateways are then using a sliding window of local
    or *ssh(1)* commands to reach the targets (if the target count per gateway
    is greater than the fanout value)

* smart display of command results (integrated output gathering, sorting by
  node, nodeset or node groups)
* standard input redirection to remote nodes
* files copying in parallel
* *pdsh* [#]_ options backward compatibility

*clush* can be started non-interactively to run a shell command, or can be
invoked as an interactive shell. Both modes are discussed here (clush-oneshot
clush-interactive).

Selecting target nodes
^^^^^^^^^^^^^^^^^^^^^^

Nodes selection
"""""""""""""""

The ``-w`` option allows you to specify remote hosts by using ClusterShell
:class:`.NodeSet` syntax, including the node groups *@group* special syntax
(cf. nodeset-groupsexpr) and the Extended String Patterns syntax (see
:ref:`class-NodeSet-extended-patterns`) to benefits from :class:`.NodeSet`
basic arithmetics (like ``@Agroup&@Bgroup``). Additionally, the ``-x`` option
allows you to exclude nodes from remote hosts list (the same NodeSet syntax
can be used here). Nodes exclusion has priority over nodes addition.

Node groups selection
"""""""""""""""""""""

For *pdsh* backward compatibility, *clush* supports two ``-g`` and ``-X``
options to respectively select and exclude nodes group(s), but only specified
by omitting any *"@"* group prefix (see example below). In general, though, it
is advised to use the *@*-prefixed group syntax as the non-prefixed notation
is only recognized by *clush* but not by other tools like *nodeset*.

For example::

    $ clush -g rhel6 cat /proc/loadavg
    node26: 0.02 0.01 0.00 1/202 23033

    $ clush -w @rhel6 cat /proc/loadavg
    node26: 0.02 0.01 0.00 1/202 23042

.. _clush-all-nodes:

All nodes selection
"""""""""""""""""""

Finally, a special option ``-a`` (without argument) can be used to select
"all" nodes, in the sense of ClusterShell node groups (see
:ref:`node groups configuration <groups-config>` for more details on special
**all** external shell command upcall).  If not properly configured, the
``-a`` option may lead to a runtime error like::

    clush: External error: Not enough working external calls (all, or map +
    list) defined to get all node


.. _clush-tree:

Tree mode
^^^^^^^^^

ClusterShell Tree mode is a major horizontal scalability improvement by
enabling a hierarchical command propagation scheme. This section describes how
to use the tree mode with the *clush* command.

.. _clush-tree-enabling:

Enabling tree mode
""""""""""""""""""

Since version 1.7, the tree mode is enabled by default when a configuration
file is present. The system-wide library configuration file
*/etc/clustershell/topology.conf* defines the routes of default command
propagation tree. When this file exists, *clush* will use it by default for
target nodes that are defined there. This file path can be changed at runtime
using the ``--topology`` command line option.

.. _clush-tree-options:

Tree-related options
""""""""""""""""""""

The ``--remote=yes|no`` command line option controls the remote execution
behavior:

* Default is **yes**, that will make *clush* establish connections up to the
  leaf nodes using a *distant worker* like *ssh*.
* Changing it to **no** will make *clush* establish connections up to the leaf
  parent nodes only, then the commands are executed locally on the gateways
  (like if it would be with ``--worker=exec`` on the gateways themselves).
  This execution mode allows users to schedule remote commands on gateways
  that take a node as an argument. On large clusters, this is useful to spread
  the load and resources used of one-shot monitoring, IPMI, or other commands
  on gateways. A simple example of use is::

      $ clush -w node[100-199] --remote=no /usr/sbin/ipmipower -h %h-ipmi -s

  This command is also valid if you don't have any tree configured, because
  in that case, ``--remote=no`` is an alias of ``--worker=exec`` worker.

The ``--grooming`` command line option allows users to change the grooming
delay (float, in seconds). This feature allows gateways to aggregate responses
received within a certain timeframe before transmitting them back to the root
node in a batch fashion. This contributes to reducing the load on the root
node by delegating the first steps of this CPU intensive task to the gateways.

.. _clush-oneshot:

Non-interactive (or one-shot) mode
""""""""""""""""""""""""""""""""""

When *clush* is started non-interactively, the command is executed on the
specified remote hosts in parallel (given the current *fanout* value and the
number of commands to execute (see *fanout* library settings in
:ref:`class-Task-configure`).

Output gathering options
""""""""""""""""""""""""

If option ``-b`` or ``--dshbak`` is specified, *clush* waits for command
completion while displaying a progress indicator (unless ``-q, --quiet``
switch is provided) and then displays gathered output results. If standard
output is redirected to a file, *clush* detects it and disable any progress
indicator.

The following is a simple example of *clush* command used to execute ``uname
-r`` on *node40*, *node41* and *node42*, wait for their completion and finally
display digested output results::

    $ clush -b -w node[40-42] uname -r
    ---------------
    node[40-42]
    ---------------
    2.6.35.6-45.fc14.x86_64


It is common to cancel such command execution because a node is hang. When
using *pdsh* and *dshbak*, due to the pipe, all nodes output will be lost,
even if all nodes have successfully run the command. When you hit CTRL-C with
*clush*, the task is canceled but received output is not lost::

    $ clush -b -w node[1-5] uname -r
    Warning: Caught keyboard interrupt!
    ---------------
    node[2-4] (3)
    ---------------
    2.6.31.6-145.fc11
    ---------------
    node5
    ---------------
    2.6.18-164.11.1.el5
    Keyboard interrupt (node1 did not complete).

Performing *diff* of cluster-wide outputs
"""""""""""""""""""""""""""""""""""""""""

Since version 1.6, you can use the ``--diff`` *clush* option to show
differences between common outputs. This feature is implemented using `Python
unified diff`_. This special option implies ``-b`` (gather common stdout
outputs) but you don't need to specify it. Example::

    $ clush -w node[40-42] --diff dmidecode -s bios-version
    --- node[40,42] (2)
    +++ node41
    @@ -1,1 +1,1 @@
    -1.0.5S56
    +1.1c

A nodeset is automatically selected as the "reference nodeset" according to
these criteria:

#. lowest command return code (to discard failed commands)
#. largest nodeset with the same output result
#. otherwise the first nodeset is taken (ordered (1) by name and (2) lowest range indexes)

Standard input bindings
"""""""""""""""""""""""

Unless option ``--nostdin`` is specified, *clush* detects when its standard
input is connected to a terminal (as determined by *isatty(3)*). If actually
connected to a terminal, *clush* listens to standard input when commands are
running, waiting for an Enter key press. Doing so will display the status of
current nodes. If standard input is not connected to a terminal, and unless
option ``--nostdin`` is specified, *clush* binds the standard input of the
remote commands to its own standard input, allowing scripting methods like::

    $ echo foo | clush -w node[40-42] -b cat
    ---------------
    node[40-42]
    ---------------
    foo

Another stdin-bound *clush* usage example::

    $ ssh node10 'ls /etc/yum.repos.d/*.repo' | clush -w node[11-14] -b xargs ls
    ---------------
    node[11-14] (4)
    ---------------
    /etc/yum.repos.d/cobbler-config.repo

.. _clush-interactive:

Interactive mode
^^^^^^^^^^^^^^^^

If a command is not specified, *clush* runs interactively. In this mode,
*clush* uses the *GNU readline* library to read command lines from the
terminal. *Readline* provides commands for searching through the command
history for lines containing a specified string. For instance, you can type
*Control-R* to search in the history for the next entry matching the search
string typed so far.

Single-character interactive commands
"""""""""""""""""""""""""""""""""""""

*clush* also recognizes special single-character prefixes that allows the user
to see and modify the current nodeset (the nodes where the commands are
executed). These single-character interactive commands are detailed below:

+------------------------------+-----------------------------------------------+
| Interactive special commands | Comment                                       |
+==============================+===============================================+
| ``clush> ?``                 | show current nodeset                          |
+------------------------------+-----------------------------------------------+
| ``clush> +<NODESET>``        | add nodes to current nodeset                  |
+------------------------------+-----------------------------------------------+
| ``clush> -<NODESET>``        | remove nodes from current nodeset             |
+------------------------------+-----------------------------------------------+
| ``clush> !<COMMAND>``        | execute ``<COMMAND>`` on the local system     |
+------------------------------+-----------------------------------------------+
| ``clush> =``                 | toggle the ouput format (gathered or standard |
|                              | mode)                                         |
+------------------------------+-----------------------------------------------+

To leave an interactive session, type ``quit`` or *Control-D*. As of version
1.6, it is not possible to cancel a command while staying in *clush*
interactive session: for instance, *Control-C* is not supported and will abort
current *clush* interactive command (see `ticket #166`_).

Example of *clush* interactive session::

    $ clush -w node[11-14] -b
    Enter 'quit' to leave this interactive mode
    Working with nodes: node[11-14]
    clush> uname
    ---------------
    node[11-14] (4)
    ---------------
    Linux
    clush> !pwd
    LOCAL: /root
    clush> -node[11,13]
    Working with nodes: node[12,14]
    clush> uname
    ---------------
    node[12,14] (2)
    ---------------
    Linux
    clush> 

The interactive mode and commands described above are subject to change and
improvements in future releases. Feel free to open an enhancement `ticket`_ if
you use the interactive mode and have some suggestions.

File copying mode
^^^^^^^^^^^^^^^^^

When *clush* is started with  the ``-c``  or  ``--copy``  option, it will
attempt to copy specified file and/or directory to the provided target cluster
nodes. If the ``--dest`` option is specified, it will put the copied files
or directory there.

Here are some examples of file copying with *clush*::

    $ clush -v -w node[11-12] --copy /tmp/foo
    `/tmp/foo' -> node[11-12]:`/tmp'

    $ clush -v -w node[11-12] --copy /tmp/foo /tmp/bar
    `/tmp/bar' -> aury[11-12]:`/tmp'
    `/tmp/foo' -> aury[11-12]:`/tmp'

    $ clush -v -w node[11-12] --copy /tmp/foo --dest /var/tmp/
    `/tmp/foo' -> node[11-12]:`/var/tmp/'

Reverse file copying mode
^^^^^^^^^^^^^^^^^^^^^^^^^

When *clush* is started with the ``--rcopy`` option, it will attempt to
retrieve specified file and/or directory from provided cluster nodes. If the
``--dest`` option is specified, it must be a directory path where the files
will be stored with their hostname appended. If the destination path is not
specified, it will take the first file or dir basename directory as the local
destination, example::

    $ clush -v -w node[11-12] --rcopy /tmp/foo
    node[11-12]:`/tmp/foo' -> `/tmp'

    $ ls /tmp/foo.*
    /tmp/foo.node11  /tmp/foo.node12

Worker selection
^^^^^^^^^^^^^^^^

By default, *clush* is using the default library worker configuration when
running commands or copying files. In most cases, this is *ssh* (See
:ref:`task-default-worker` for default worker selection). ClusterShell
supports other worker types like *rsh* or also *pdsh*. This worker selection
could be changed at runtime thanks to ``--worker`` command line option::

    $ clush -w node[11-12] --worker=rsh echo ok
    node11: ok
    node12: ok



.. [#] LLNL parallel remote shell utility
   (https://computing.llnl.gov/linux/pdsh.html)

.. _seq(1): http://linux.die.net/man/1/seq
.. _Python unified diff:
   http://docs.python.org/library/difflib.html#difflib.unified_diff

.. _ticket #166: https://github.com/cea-hpc/clustershell/issues/166
.. _ticket: https://github.com/cea-hpc/clustershell/issues/new
