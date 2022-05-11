# Cylc 8 sub-workflow example

A sub-workflow is a workflow that runs as a task in another workflow.

This example employs a set of reusable scripts to make managing sub-workflows
easier.


## Why use sub-workflows?

The structure of a Cylc workflow is determined by the `flow.cylc` file at
start-up. The workflow can take different paths through the graph depending on
run time events, but all the paths must be defined at start-up.
 
Sub-workflows are useful when the internal structure of a sub-graph can only
be determined at run time. Running the sub-graph as a separate workflow means
we can configure it anew each time we create and run a new instance.

For example, consider a weather forecasting system that in each forecast cycle
needs to run multiple local extreme-weather models (and associated processing)
the number and location of which depends on the current situation. This can be
done by launching a dynamically determined
number of sub-workflows with
dynamically determined input parameters to set their locations and so on, in
each forecast cycle.


## Things to be aware of


### Sub-workflow locations and installation

Sub-workflows should be defined in sub-directories of the main workflow source
directory (to the main workflow, they are just tasks).

After installing the main workflow, the installed sub-workflow definition
is the source for creating sub-workflow instances on the fly, at run time.


### Sub-workflows at run time

A sub-workflow is just a normal workflow, with its own ID and run directory. It
just happens to be launched by a task in another workflow.

To the main workflow, a sub-workflow is a monolithic task. To see the internal
tasks you have to view the sub-workflow itself.

Workflows can potentially finish "successfully" without reaching their intended
end point, e.g. in response to a stop command. Sub-workflow launcher tasks need
to detect this and interpret it as failure.

You can manipulate a running sub-workflow, e.g. to retrigger failed tasks,
directly via its workflow ID. But to **restart** or **rerun** a stopped
sub-workflow, retrigger its launcher task. (If you run an application
independently of a workflow that also happens to run it, you can't expect the
workflow to know you did that).

However, if a direct restart did happen (e.g. during workflow auto-migration)
just re-trigger the launcher task (with `SUBWF_RERUN_FROM_SCRATCH=false`)
after the sub-workflow has finished. The sub-workflow will restart again and
exit immediately with success, when it sees there's nothing more to do.

If a sub-workflow stalls after unexpected task failures, the launcher task will
be stuck as running. To avoid this, configure sub-workflows to abort on a stall
timeout. The timeout should allow time to intervene, e.g. by retriggering
failed tasks, on restart, otherwise (short of `cylc play --pause`) it will just
shut down again right away.

Sub-workflows present a different housekeeping problem than ordinary tasks,
because each instance generates a new run directory. These can be removed with
`cylc clean` after extracting any important results.

Sub-workflows can be detaching (the launcher task runs `cylc play <SUBWF_ID>`)
or non-detaching (the launcher task runs `cylc play --no-detach <SUBWF_ID>`)

- Non-detaching sub-workflows have their status mirrored instantly in the
launcher task, and their stdout appears in its job log, but they are not
compatible with scheduler host load balancing and workflow auto migration.

- Detaching sub-workflows have delayed status updates polled by the launcher
task, but they are compatible with run host load balancing and workflow auto
migration.


## The example in detail

### Main source directory 

The main workflow source directory, with nested sub-workflow source directory:

```bash
cylc-subwf-example> tree $PWD
/home/oliverh/cylc-src/cylc-subwf-example
├── main  # <--- main workflow source directory
│   ├── bin  # <--- workflow bin scripts
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

### Install the main workflow

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

### Run the main workflow

```bash
main> cylc play --no-detach main
...
(DONE)
```

The sub-workflow launcher task definition in the main workflow looks like this:
```ini
[runtime]
    [[run-sub]]
        script = subworkflow-run sub 1/post
        err-script = subworkflow-err sub
```

Every instance of `run-sub` calls `subworkflow-run` to install and run a new
instance of `sub`, and expects the final task `1/post` to succeed before shut
down.

The `subworkflow-err` script kills detached sub-workflow instances if the
launcher task gets killed.

Sub-workflow names are based on the parent workflow and cycle point, to group
parent and sub-workflows together, and to avoid run-directory clashes.

For example, in cycle point `1` the task `1/run-sub` in `main/run2` installs
and runs the sub-workflow instance `main-run2-c1-sub/run1`. Flat sub-workflow
names are ~necessary because we don't allow `runN` as an internal path component.

Whether a sub-workflow detaches or not is controlled by a shell variable in the
launcher task job environment:
```bash
# launcher task environment, main workflow:
SUBWF_DETACH=false  # (DEFAULT) run sub-workflow in the launcher job script
# or:
SUBWF_DETACH=true  # detach the sub-workflow from the launcher job script
```

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

### Hierarchical main workflow names

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

### Stopping or killing sub-workflows

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

### Updating the installed sub-workflow definition

To update the installed sub-workflow definition from source, just reinstall the
main workflow (the same as you would to update any main workflow task):
```bash
main> cylc reinstall main
REINSTALLED main/run1 from /home/oliverh/cylc-src/cylc-subwf-example/main
```

The updated definition will apply for all future sub-workflow instances. It will
also affect already-installed but stopped sub-workflow instances restarted by
retriggering their launcher task, because the `subworkflow-run` script uses
`cylc install` to create a new sub-workflow run directory for a rerun from
scratch, and `cylc reinstall` to update an existing sub-workflow run directory
for a restart.


### Restarting or rerunning a stopped sub-workflow

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


### Failed launcher task: failed to submit sub-workflow

If the launcher task itself is broken, check its job logs to diagnose the problem.
Then fix the bug in the source directory, reinstall the main workflow, and
retrigger the launcher task.

(i.e., just like any other main workflow task fix).


### Failed launcher task: sub-workflow failed

If the sub-workflow failed, check the sub-workflow logs to diagnose the problem.
Fix the sub-workflow definition in the source directory, reinstall the main
workflow, then restart or rerun the sub-workflow by retriggering the launcher.


### Failed tasks in a still-running sub-workflow

If there are unexpected task failures in a sub-workflow that hasn't yet
aborted on the stall timeout, the easiest thing to do is stop it a little
early and proceed as for a failed sub-workflow (above).

Bu, if you want to you can manually reinstall and reload the sub-workflow
instance like this:

```bash
main> cylc reinstall main  # <--- update the sub-workflow defn from source
REINSTALLED main/run1 from /home/oliverh/cylc-src/cylc-subwf-example/main

main> cylc reinstall main-run1-c1-sub  # <--- update the instance
REINSTALLED main-run1-c1-sub/run1 from /home/oliverh/cylc-run/main/sub

main> cylc reload main-run1-c1-sub  # <--- (if the flow.cylc changed)
```

(No need to retrigger the launcher task now; it will remain in the running
still throughout this procedure).


*** Recovering from workflow automigration

If sub-workflows are run in detaching mode, they can be auto-migrated.

This means they will shut down early and restart themselves on another run
host directly, i.e. without going via the launcher task in the main workflow.
(A sub-workflow does not know its a sub-workflow!).

Depending on the polling interval, the main workflow may detect the early
shutdown as failure.

Furthermore, even if the main workflow ...

If the main workflow gets auto-migrated while the launcher task is still in the
running state, it will poll the launcher on restart but will detect failure
again if the sub-workflow has migrated in the meantime because its .


