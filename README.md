## taskwarrior `.taskrc` / vit `.vitrc/config.ini`

I use `vit`/`task` for long-term tracking of ideas which turn into projects/goals and grow more and more tasks. It's a permanent storage archive that feeds into my paper-based daily bullet journalling which I use to organize my days.

### feature list

* idea / project / phase / task workflow
  * idea -> project/goal
    * [X] undeveloped ideas are tasks with an `+idea` tag. they don't need to be specific, measurable goals
    * [ ] how will I add annotations to an idea to turn it into a concrete project/goal?
    * [X] dedicated `ideas` report shows ideas/projects in all states, with their annotations
  * project/goal -> phase/sprint
    * [X] `sprint` report for looking at all *pending* tasks of a specific project (with clear dependencies?)
    * [X] `project` report for looking at *all* tasks of a project, including waiting ('someday') tasks AND completed tasks at bottom!
    * [X] use `wait:someday` vs no wait attribute to distinguish next-sprint vs. non-sprint tasks within a project
  	* [ ] find a way to apply `+sprint` to all next-sprint (i.e. non-waiting) tasks in the current project view
* daily/urgent tasks in the `next` report
  * urgency/priority calculation
    * [Bulletproof tasks](https://www.notion.vip/bulletproof-tasks/) formula calculation is lower value -> higher priority/urgency
      * [Formula](https://www.notion.so/Bulletproof-Task-Formulas-5be62765ea5b465cb9a2dc38b950d8a5): `Bulletproof Urgency = ( Pri(Hi=2/4/Lo=6/Someday=8) + Imp(Hi=2/4/Lo=6) + Work(Hi=1/2/Lo=3) + Enjoyment(Hi=3/2/Lo=1) ) / .06`
    * [X] display `impact`, `workload` and `enjoyment` udas
    * [ ] add shortcuts for changing `impact`, `workload` and `enjoyment`
* [X] start/stop Clockify time entries for specific tasks from within vit (via `clockify-cli`)
* review
  * built-in static reports: `task burndown.daily` / `task burndown` / `task timesheet`
  * [ ] try out `tasksh review`

### resulting task pipeline:

1. `project` report which shows all tasks for a project (including completed, at the bottom)
2. `sprint` report which shows all pending or waiting task for a project (but not completed)
3. `next` report shows all due, active, `+next` and `+sprint` tasks

### keybindings

#### important default bindings

* `a` add task
* `b` begin task (set `start:now` which makes the task `+ACTIVE`)
* `d` mark task done (`+COMPLETED`)
* `u` undo last action

#### custom bindings

##### report navigation

* `n` to jump to `:next` report
* `P` to open `:project` report, waiting to fill in project to filter for
* `I` `:ideas` report (shows active and inactive `+idea` tasks)

##### task manipulation

* `A` add a new *active* task (`start:now `)
* `W` add a new *weekly* task (`due:eow +weekly `)
* `i` add new task with tag `+idea`
* `w` set task `wait:someday`
* `s` set task `+sprint`
* `O` adds anotation

##### other

* `c` starts a Clockify time entry via `clockify-cli in {TASK_DESCRIPTION}`
* `C` sets `start:now` AND starts a Clockify time entry
* `S` remove `wait` (set `wait:`)

<!-- previous:
`clockify-cli out`
-->

### taskwarrior virtual tag logic

Information about this is a bit scattered in the docs, but the two main sources are:
* https://taskwarrior.org/docs/using_dates.html
* https://taskwarrior.org/docs/terminology.html

#### Mandatory virtual tag sets

##### Task status virtual tags

The following 4 virtual tags are mutually exclusive, and any task will have one of them:

* `COMPLETED` (has an `end` date)
* `DELETED` (has an `end` date)
* `WAITING` (has a `wait` date. the `wait` entry is automatically removed when the `wait` date is reached)
* `PENDING` for any task for which none of the above apply (`INCOMPLETE` would've been a more intuitive label tbh)
It's noteworthy that both `PENDING` *and* `WAITING` tasks can be assigned `ACTIVE`! Only completed and deleted tasks cannot be started.

(There is also the `RECURRING` tag for recurring events which is not covered here.)

##### Task dependency virtual tags

The following 2 virtual tags are mutually exclusive, and any task will have one of them:

* `BLOCKED` if there is at least one incomplete(?) depending task
* `UNBLOCKED` otherwise

#### Optional virtual tags

##### Task readiness 

* `SCHEDULED` if the task has a `scheduled` date (whether it is in the past or the future)
Note that this virtual tag is not mutually exclusive with:
* `READY` if the task is both `PENDING` and `UNBLOCKED` and has either no `scheduled` date or the `scheduled` date is in the past

##### Task activity

* `ACTIVE` if the task has a `start` date

Both the date (and with it the virtual tag) are removed when a task is completed or deleted. However, a currently active task maintains its `ACTIVE` state when it is assigned a `wait` date (and thus changes to a `WAITING` task status).

##### Task dueness

Like with `ACTIVE`, all of the following tags are only applied to `PENDING` and `WAITING` tasks.

When a task has a `due` date it is given the following virtual tags if the due date is within the same named time unit as 'now' *independently of whether the actual due date is in the past or the future from now*: `YEAR`, `QUARTER`, `MONTH`, `WEEK`, `TOMORROW`, `TODAY`, `DUETODAY` (not sure what's the difference there?), `YESTERDAY`

There are also two mutually exclusive tags:
* `OVERDUE` if the due date is in the past
* `DUE` if the due date lies between 'now' and midnight 6 days from now

##### Task expiry

Any task with an `until` date carries the `UNTIL` virtual tag, which seems to not interact with the any of the other tags. Upon reaching the `until` date, the task is automatically deleted (not sure what happens with the `until` date or the tag at this point).
