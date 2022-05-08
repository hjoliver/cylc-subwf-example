# Cylc 8 sub-workflow example

A sub-workflow is a workflow that runs as a task in another workflow.

## Why use sub-workflows?

The structure of a workflow is determined at start-up. The path taken through
the graph can vary at run time, but the paths must all be known at start-up.
 
Sub-workflows are useful when the internal structure of a sub-graph can only
be determined at run time. Running the sub-graph as a separate workflow means
we can configure it anew each time we create and run a new instance.

For example, consider a weather forecasting system that in each forecast cycle
needs to run multiple local extreme-weather models (and associated processing)
the number and location of which depends on the current situation. This can be
done by launching a dynamically determined number of sub-workflows with
dynamically determined input parameters to set their locations and so on.

## How it works

This working example includes a set of reusable scripts to make launching and
managing sub-workflows easier. It works as follows:

Automatic sub-workflow naming based on the parent workflow and cycle point,
  for easy CLI targetting of parent- and sub-workflows together, and to avoid
  run-directory clashes.
```bash
# Main workflow runs, after "cylc install main":
main/run1/  # <--- run1 of main
main/run2/  # <--- run2 of main (after re-installation to run again)

# Sub workflow runs, after running main/run1 and main/run2:
main-run1-c1-sub/run1/  # <--- run1 of sub by main/run1 in cycle 1
main-run1-c2-sub/run1/  # <--- run1 of sub by main/run1 in cycle 2

main-run2-c1-sub/run2/  # <--- run2 of sub by main/run2 in cycle 1
```
(Flat sub-workflow names are necessary because we don't allow `runN` as an
internal path component).

This works for hierarchical main workflow names too, to allow group collapse in
the UI scan sidebar:
```bash
# Main workflow runs, after "cylc install hydro/main":
hydro/main/run1/  # <--- run1 of hydro/main
hydro/main/run2/  # <--- run2 of hydro/main

# Sub workflow runs, after running hydro/main/run1:
hydro/main-run1-c1-sub/run1/  # <--- run1 of sub, by hydro/main/run1 cycle 1
...
```

Check that sub-workflows reached the desired end point when they stopped.
```bash
# launcher task, main worklow
script = subworkflow-run sub 1/post  # <--- sub-workflow NAME and FINAL TASK
```

Re-start OR re-run sub-workflows from scratch, by retriggering launcher tasks.
```bash
# launcher task environment, main workflow:
SUBWF_RERUN_FROM_SCRATCH=false  # (DEFAULT) re-start an existing run if retriggered
# or:
SUBWF_RERUN_FROM_SCRATCH=true  # install and run again from scratch if retriggered
```
(Use `cylc broadcast` to set `SUBWF_RERUN_FROM_SCRATCH=true` before retriggering the
launcher task.)

Subworkflows can be **non-detaching** (with status instantly mirrored in the
  launcher tasks, but not compatible with scheduler host load balancing and
  workflow auto migration) **or detaching** (with delayed status updates polled
  by the launcher task, which is compatible...).
```bash
# launcher task environment, main workflow:
SUBWF_DETACH=false  # (DEFAULT) run sub-workflow in the launcher job script
# or:
SUBWF_DETACH=true  # detach the sub-workflow from the launcher job script
SUBWF_PING_INTERVAL=20 # (secs, default 30) check detached sub-wf is running
```

## Things to be aware of

Technically, a sub-workflow is just a normal workflow with its own ID and run
directory. It just happens to be launched by a task in another workflow.

To the main workflow, a sub-workflow is just a task. Its status (running,
succeeded, failed, etc.) reflects that of the sub-workflow, but to see the
internal tasks we have to look at the sub-workflow itself.

A workflow can finish "successfully" without reaching its intended end point,
e.g. in response to a stop command, So a launcher task needs to check for the
end point once its subworkflow finishes.

Sub-workflows can be manipulated directly via their own workflow ID, e.g.
to fix and retrigger failed tasks.

Sub-workflows should be re-started by retriggering the launcher task, not
directly, because the main workflow job wrapper is needed for the completion
status to propagate up.

If a direct re-start was done, however (e.g. by the auto-migration process)
just re-trigger the launcher task (with `SUBWF_RERUN_FROM_SCRATCH=true`) *after
the sub-workflow has finished* to update its status. The sub-workflow will
restart and exit immediately with success, because it already finished.

If a sub-workflow stalls as a result of unexpected task failures the launcher
task will appear stuck as running. So sub-workflows should be configured to
abort on a stall timeout, with enough time to intervene after restarting in
the stalled state (otherwise you'll have to start it with `--pause`).

Sub-workflows should be defined in sub-directories of the main workflow source
directory (they are just tasks to the main workflow, after all).

When the main workflow is installed, its installed copy of the sub-workflow
definition is used as the source for creating (by installing to new run
directories) sub-workflow instances on the fly, at run time.

Sub-workflows present a different housekeeping problem than other main workflow
tasks, because each instance generates a new run directory. To keep things
tidy, a main workflow task should retrieve results from the finished
sub-workflow (if it did not write them directly to the main workflow run
directory) before removing it with `cylc clean`.


## FAQ

**How do I determine the cause of a sub-workflow launcher task failure?**

This means the sub-workflow failed to complete successfully, so check the
sub-workflow scheduler log and job logs.


**How do I stop (etc.) a main workflow and its sub-workflows all at once?**
```console
$ cylc stop 'main*'  # <--- catches main/run* and main-run1-c1-sub/run*
$ cylc clean 'main*'
```

**How do I update and re-start a sub-workflow instance that stopped early?**
```console
# Update the sub-workflow definition in the main source directory, then:
$ cylc reinstall main/run1  # <--- update the installed main workflow
$ cylc reinstall main-run1-c1-sub  # <--- update the installed sub-workflow instance
$ cylc trigger main/run1//1/run-sub  # <--- re-trigger the sub-workflow launcher
```

**How do I update and re-run a sub-workflow instance from scratch?**
```console
# (reinstall as above, if needed)
# Tell the launcher task to re-install for a new run from scratch:
$ cylc broadcast -n run-sub -p 1 -s '[environment]SUBWF_RERUN_FROM_SCRATCH=true' main/run1
$ cylc trigger main/run1//1/run-sub  # <--- re-trigger the launcher task
```

**How do I kill a sub-workflow?**
```console
# 1. kill the launcher task (this kills its sub-workflow too):
$ cylc kill main//1/run-sub

# 2. OR stop or kill the sub-workflow (will be detected by the launcher task):
$ cylc stop main-run1-c1-sub
  # or:
$ kill PID  # process ID of the sub-workflow scheduler
```
