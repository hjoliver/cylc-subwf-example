# Cylc 8 sub-workflow example

A sub-workflow is a workflow that is run by a task in another workflow.

Cylc does not have built-in support for sub-workflows, but a Cylc task can run
any application, including another instance of the Cylc scheduler.

This example includes a set of reusable scripts to make sub-workflows easy.


## Why use sub-workflows?

The structure of a Cylc workflow is determined at start-up, when the scheduler
parses the `flow.cylc` file. Different paths can be taken through the graph at
run time, but the paths must be known at the outset.
 
Sub-workflows are useful when the internal structure of a sub-graph can only
be determined at run time. Running the sub-graph as a separate workflow means
we can configure it anew each time we need a new instance.

For example, consider a weather forecasting system that in each cycle needs to
run multiple local extreme-weather models (and associated processing), the
number and location of which depends on the current situation. This can be
done by launching a dynamically determined number of sub-workflows configured
by dynamically determined input parameters for location and so on, in each
cycle.


## Understanding sub-workflows

### Sub-workflow source directories

Sub-workflows should be defined in a sub-directory of the main workflow source
directory (just like other applications that the main workflow runs).


### Sub-workflow installation

On installing the main workflow to a run directory, the installed sub-workflow
definition becomes the source for creating sub-workflow instances on the fly at
run time (just as other installed task definitions are templates for creating
task instances at run time).

Unlike other task instances, however, sub-workflow instances also have to be
installed to their own run directories in order to run. This is handled
automatically by the `subworkflow-run` script called by the launcher task in
the main workflow.


### Sub-workflows are just normal workflows

A sub-workflow instance is a normal workflow with its own ID and run directory.
It just happens to be installed and run by a task in another main workflow.

You can manipulate a running sub-workflow, e.g. to retrigger failed tasks in it,
directly via its own workflow ID. To see the individual tasks in the
sub-workflow, you have to view the sub-workflow itself.


### Sub-workflow completion

Any workflow can potentially finish "successfully" without reaching its
intended end point, e.g. in response to a stop command. Sub-workflow launcher
tasks need to detect this and interpret it as failure. The easiest way to
do this currently is to check that the `task_pool` table in the sub-workflow
run database is empty. An early shutdown, whether under an error condition or
not, will leave entries in the task pool table to continuing the workflow after
a restart.

### Sub-workflow stall

If a sub-workflow stalls after unexpected internal task failures, the main
workflow's launcher task will appear to be stuck as running. To avoid this,
configure sub-workflows to abort on a stall timeout, which will show up as a
failed launcher task (correctly: the sub-workflow aborted without completing
successfully). The stall timeout interval should be sufficient to allow
intervention after restarting the sub-workflow (otherwise you'll have to
restart it with `--pause`).


### Sub-workflow restart or rerun

A sub-workflow instance should not be restarted or rerun directly (i.e., via
its workflow ID) because the main workflow won't see it. (If an application
happens to be used in a workflow and you run it independently, you can't
expect the workflow to know you did that). However, it is easy enough to update
status after a direct restart if necessary - see *Recovering from auto
migration* below.

Instead, retrigger the main launcher task. By default this will restart its
sub-workflow, but you can also choose to rerun it from scratch (see below). 


### Detaching and non-detaching sub-workflows

Sub-workflows can be:
- Detaching (the launcher task runs `cylc play <SUBWF_ID>`)
  - these have delayed status updates polled by the launcher task, but they are
    compatible with scheduler run host load balancing and workflow auto migration.
- Non-detaching (the launcher task runs `cylc play --no-detach <SUBWF_ID>`)
  - these have status mirrored instantly in the launcher task, and their stdout
    appears in its job log, but they are not compatible with scheduler run host
    load balancing and workflow auto migration.

In this example, detaching is controlled by a shell variable used in the
`subworkflow-run` script:
```bash
# launcher task environment, main workflow:
SUBWF_DETACH=false  # (DEFAULT) run sub-workflow in the launcher job script
# or:
SUBWF_DETACH=true  # detach the sub-workflow from the launcher job script
```

### Sub-workflow housekeeping

Sub-workflows present a different housekeeping problem than ordinary tasks,
because each instance generates a new run directory. These can be removed with
`cylc clean` after extracting any important results.


## The example in detail

#### Main workflow source directory 

```bash
cylc-subwf-example> tree $PWD
/home/oliverh/cylc-src/cylc-subwf-example
├── main  # <--- main workflow source directory
│   ├── bin  # <--- main workflow bin scripts
│   │   ├── subworkflow-clean
│   │   ├── subworkflow-err
│   │   ├── subworkflow-kill
│   │   ├── subworkflow-lib
│   │   └── subworkflow-run
│   ├── flow.cylc  # <--- main workflow config file
│   └── sub  # <--- sub-workflow source directory
│       ├── bin  # <--- sub-workflow bin scripts
│       │   └── ...
│       └── flow.cylc  # <--- sub-workflow config file
└── README.md
```

(Note the names "main" and "sub" are arbitrary).


#### Install the main workflow

```bash
cylc-subwf-example> cd main/
main> cylc install
INSTALLED main/run1 from /home/oliverh/cylc-src/cylc-subwf-example/main

main> tree ~/cylc-run/main
/home/oliverh/cylc-run/main
├── _cylc-install
│   └── source -> /home/oliverh/cylc-src/cylc-subwf-example/main
├── run1  # <--- main worklow run directory
│   ├── bin
│   │   └── ...
│   ├── flow.cylc
│   ├── ...
│   └── sub   # <--- source for creating sub-workflow instances
│       ├── bin
│       │   └── ...
│       └── flow.cylc
└── runN -> run1
```

#### Run the main workflow

```bash
> cylc play --no-detach main
...
(DONE)
```

The launcher task definition in the main workflow looks like this:
```ini
[runtime]
    [[run-sub]]
        script = subworkflow-run sub
        err-script = subworkflow-err sub
```

Every instance of `run-sub` calls `subworkflow-run` to install and run a new
instance of `sub`.

The `subworkflow-err` script kills detached sub-workflow instances if the
main launcher task gets killed.

Sub-workflow names are based on the parent workflow and cycle point, to group
parents and sub-workflows together, and to avoid run-directory clashes.

For example, the task instance `1/run-sub` in `main/run8` installs and runs the
sub-workflow instance `main-run8-c1-sub/run1`. (Flat sub-workflow names are
~necessary because we don't allow `runN` as an internal path component.)


#### Sub-workflow run directories

Check the run directories after running the main workflow. Note that the
sub-workflow source links point to the main run directory, not the main source
directory:

```bash
> tree -d -L 3 ~/cylc-run
/home/oliverh/cylc-run
├── main  # <--- main workflow 
│   ├── _cylc-install
│   │   └── source -> /home/oliverh/cylc-src/cylc-subwf-example/main
│   ├── run1  # run1 of main
│   │   ├── bin
│   │   ├── log
│   │   ├── share
│   │   ├── sub
│   │   └── work
│   └── runN -> run1
├── main-run1-c1-sub  # <--- sub-workflow of main/run1 at cycle 1
│   ├── _cylc-install
│   │   └── source -> /home/oliverh/cylc-run/main/run1/sub
│   ├── run1  # run1 of main-run1-c1-sub
│   │   ├── bin
│   │   ├── log
│   │   ├── share
│   │   └── work
│   └── runN -> run1
└── main-run1-c2-sub  # <--- sub-workflow of main/run1 at cycle 2
    ├── _cylc-install
    │   └── source -> /home/oliverh/cylc-run/main/run1/sub
    ├── run1  # run1 of main-run1-c2-sub
    │   ├── bin
    │   ├── log
    │   ├── share
    │   └── work
    └── runN -> run1
```

#### Hierarchical workflow names

This naming convention works for hierarchical main workflow names too:
```bash
# Main workflow runs, after "cylc install hydro/main":
hydro/main/run1/  # <--- run1 of hydro/main
hydro/main/run2/  # <--- run2 of hydro/main
...
# Sub workflow runs, after running hydro/main/run1:
hydro/main-run1-c1-sub/run1/  # <--- run1 of sub, by hydro/main/run1 cycle 1
...
```

#### Stopping or killing sub-workflows

Stopping a sub-workflow early is equivalent to killing it - the launcher task
in the main workflow will (correctly) detect failure to complete.

```bash
# Killing a launcher task stops its sub-workflow:
cylc kill main//1/run-sub

# Stopping (or killing) the sub-workflow early is detected by the launcher:
cylc stop --now main-run1-c1-sub

# Use shell globbing to stop main and sub-workflows at once:
cylc stop 'main*'  # <--- catches main* and main-run*
```

#### Updating installed sub-workflow definitions

To update the installed sub-workflow definition from source, reinstall the
main workflow (just as you would to update any main workflow task):
```bash
> cylc reinstall main
REINSTALLED main/run1 from /home/oliverh/cylc-src/cylc-subwf-example/main
```

The updated definition will apply for all future sub-workflow instances. It will
also affect already-installed but stopped sub-workflow instances restarted by
retriggering their launcher task, because the `subworkflow-run` script uses
`cylc install` to create a new sub-workflow run directory from the "source" in
the main run directory, for a rerun from scratch; and `cylc reinstall` to
update an existing sub-workflow run directory for a restart.


#### Restarting or rerunning stopped sub-workflows

To restart an existing stopped sub-workflow instance, just retrigger the
launcher task. This will update the existing run directory and restart it.
```bash
cylc trigger main/run1//1/run-sub
```

To rerun an existing stopped sub-workflow from scratch, tell the launcher task
you want a rerun before triggering it:
```bash
cylc broadcast -n run-sub -p 1 -s '[environment]SUBWF_RERUN_FROM_SCRATCH=true' main/run1
cylc trigger main/run1//1/run-sub
```
(Don't forget to cancel the broadcast if a subsequent restart is needed.)


#### Failed launcher task: failed to submit sub-workflow

If the launcher task itself is broken, check its job logs to diagnose the problem.
Then fix the bug in the source directory, reinstall the main workflow, and
retrigger the launcher task (just as for any other main workflow task fix).


#### Failed launcher task: sub-workflow failed

If the sub-workflow instance failed, check its logs to diagnose the problem.
Fix the sub-workflow definition in the source directory, reinstall the main
workflow, then restart or rerun the instance by retriggering the launcher.


#### Failed tasks in a still-running sub-workflow

If there are unexpected task failures in a sub-workflow that hasn't yet aborted
on the stall timeout, the easiest thing to do is stop it a little early and
proceed as for a failed sub-workflow (above).

But, if you like you can manually reinstall and reload the running sub-workflow
instance like this:

```bash
> cylc reinstall main  # <--- update the sub-workflow defn from source
REINSTALLED main/run1 from /home/oliverh/cylc-src/cylc-subwf-example/main

> cylc reinstall main-run1-c1-sub  # <--- update the instance
REINSTALLED main-run1-c1-sub/run1 from /home/oliverh/cylc-run/main/sub

> cylc reload main-run1-c1-sub  # <--- (if the flow.cylc changed)
```

(No need to retrigger the launcher task now; it will remain in the running
still throughout this procedure).


#### Recovering from workflow auto-migration

Detached workflows are subject to auto-migration: the scheduler can be told
(via global config) to shut down and restart itself on another host.

Migrated sub-workflows will not be seen by the main workflow (whether it
migrated or not) because they were not restarted by the launcher task.

To recover from this, just wait for the migrated sub-workflow to finish then
retrigger its launcher task. It will restart again then shut down immediately,
reporting success, because it already ran to completion.
