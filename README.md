# a taskwarrior / vit / vimwiki pipeline

## history

* in spring 2021 I started using [taskwarrior]() with [vit](https://github.com/vit-project/vit) as an interactive frontend
* tried out taskwiki, but I found it to be very slow and it [does not support batch updating of tasks](https://github.com/tools-life/taskwiki/issues/196#issuecomment-634789028)
* in the end I implemented a [custom script for populating/parsing tasks](bin/taskproject) which I am calling from vit. it populates a vimwiki file with task lists which I can then edit interactively in `vipe`, on exit it parses changes and writes them into the taskwarrior database before returning to vit.

## workflow

My `vit`/`task` database doubles as [both parts of task and project management](https://medium.com/strong-opinions/daily-planning-the-bulletproof-system-54367a45b422): on one hand (more unusally) a *personal dashboard* for long-term tracking of ideas which turn into projects/goals and grow more and more specific tasks, which ends up being my actual *to do list*. It's a permanent storage archive that feeds into my paper-based daily bullet journalling which I use to organize my days. I make use of the following reports:

* personal dashboard
  * `:ideas` report for dumping all sorts of ideas
  * `:projects` report for getting an overview of active projects (see below)
  * *(needs revision: `:goals` report for strategic and goal-oriented mid- to long-term planning
    * [Angel Salinas'](https://medium.com/strong-opinions/planning-ii-your-personal-dashboard-f6a9d7505f62) doesn't work that well for me, gotta look for a more suitable one*
* to-do list
  * current tasks via the `:next` report -- for just getting things done (this replaces my daily todo list/journal)
  * collecting, grouping into sprints and then actuating per-project tasks via [my custom `taskproject` python script](bin/taskproject) ~~the `:project` report (this replaces my former project journal)~~

### idea / project / phase / task workflow

* idea -> project/goal
  * [X] undeveloped ideas are tasks with an `+idea` tag. they don't need to be specific or measurable like goals. I add them as I go along.
  * [X] dedicated `:ideas` report shows ideas/projects in all states, with their annotations
  * [X] `+idea` tasks are hidden from `:next` even if they're `+ACTIVE`
  * [ ] ~~how will I add annotations to an idea to turn it into a concrete project/goal? -- look at onenote~~
  * [X] once I start working on an idea it will be tagged as `+project`, assigned a `project:` which all consequent goals/tasks will be grouped under as well, and (usually) made `+ACTIVE` by setting a `start:` date

In a previous paper-based project journal I kept track (and a check on the number) of currently active projects by grouping them in different stages. The different stages (and their corresponding taskwarrior states) are:

* ongoing (`+ACTIVE`)
* blocked (`+blocked`)
* filed (`+WAITING`)
* completed (`+COMPLETED`)
* scrapped (`+DELETED`)

### goals

* [X] specific, measureable long-term goals are tasks with a `+goal` tag. they should certainly have a `scheduled` (start) date, and optionally also a `due` date (if there is an actual deadline)
* [X] the `:goals` report shows all current and future goals sorted by their `scheduled` dates
* completed/deleted goals at the bottom
* ~~project/goal -> phase/sprint
* [X] the `:project` report lists *all* tasks of a project: `+idea`/`+goal`/`+next` first, then `+ACTIVE`, then all pending tasks by urgency, then `+WAITING` ('someday') tasks followed by completed/deleted tasks (sorted by their `end`) at bottom!
* [X] `<Space>` applies the `+next` tag to a task, marking them for the next sprint
* [X] `s` applies `waiting:someday` to a task, stashing it away (pushing it to the bottom)
* [X] `X` applies `scheduled:now` and the same `due:` to all `+next` tasks in the current project view
* [X] `:sprint` report for looking at all incomplete tasks that are marked for the next `+sprint` of a of a project~~

### first attempt: doing the project pipeline within vit itself

<s>tasks with real deadlines have a `due:` date so that's easy, but what about projects I work on for myself.

* adding a `+next` tag queues a task for the next sprint of that project
* to start a sprint, I apply `scheduled:now` to all `+next +PENDING` tasks of a project
  * being `+SCHEDULED +READY` makes those tasks show up on my `next` report, but without a due date
  * TODO I should set a deadline as well?


## taskwarrior `.taskrc` / vit `.vitrc/config.ini`


### visualizing task importance

To add some structure/differentiation to task lists, try to visually *remind myself of why I'm doing specific tasks* (because they're important, fun, etc).

Notion's [Bulletproof tasks](https://www.notion.vip/bulletproof-tasks/) uses 4 attributes, but what do those dimensions actually mean? Some extra resources:

* [Action priority Matrix](https://www.mindtools.com/pages/article/newHTE_95.htm): impact & effort (=workload)
  * high impact low effort: quick win
  * high impact high effort: major project
  * low impact low effort: fill in
  * low impact high effort: thankless task
* [7 different methods for prioritising](https://www.greycampus.com/blog/project-management/six-step-process-to-priroritize-project-tasks)
* My own definitions (in descending order of importance!):
  * objective factors
    * `impact`: whether finishing the task will make a big difference in achieving a larger objective
    * `workload`: how hard/much work it will be
  * subjective factors
    * `priority`: whether, despite having no real deadline, I subjectively want to get this done soon
    * `enjoyment`: whether I will enjoy performing the task
  * [X] vit shortcuts for changing each of the 4 attributes
    * [X] `!` for `impact`
    * [X] `@` for `workload`
    * [X] `#` for `priority`
    * [X] `$` for `enjoyment`

[Bulletproof tasks formula for calculating an overall 'Urgency'](https://www.notion.so/Bulletproof-Task-Formulas-5be62765ea5b465cb9a2dc38b950d8a5) (where lower value -> higher priority/urgency)

`Bulletproof Urgency = ( Priority(Hi=2/4/Lo=6/Someday=8) + Impact(Hi=2/4/Lo=6) + Workload(Hi=1/2/Lo=3) + Enjoyment(Hi=3/2/Lo=1) ) / .06`

Flipping the values around (so that higher number -> higher urgency) we get:
* main: higher priority tasks are more urgent, 'someday'/'waiting' tasks have lowest value
* main: higher impact tasks are more urgent
* secondary: higher workload tasks are more urgent
* secondary: lower enjoyment tasks are more urgent

Using only `H`, `L` and the initial undefined for every UDA I have the following coefficients:

```
urgency.uda.priority.L.coefficient=-2
urgency.uda.priority..coefficient=0
urgency.uda.priority.H.coefficient=2

# 'someday' tasks are marked as waiting, so they have (at least) -3 deducted by default
urgency.uda.impact.L.coefficient=-2
urgency.uda.impact..coefficient=0
urgency.uda.impact.H.coefficient=2

urgency.uda.workload.L.coefficient=-1
urgency.uda.workload..coefficient=0
urgency.uda.workload.H.coefficient=1

urgency.uda.enjoyment.L.coefficient=1
urgency.uda.enjoyment..coefficient=0
urgency.uda.enjoyment.H.coefficient=-1

# adjustments
urgency.waiting.coefficient=-5
```


* [X] start/stop Clockify time entries for specific tasks from within vit (via `clockify-cli`)
* review
  * built-in static reports: `task burndown.daily` / `task burndown` / `task timesheet`
  * [ ] try out `tasksh review` -- requires uda?

### resulting task pipeline

The `:next` report shows 3 types of tasks:
1. first are `+ACTIVE` tasks (but `-idea -goal`), for quickly adding stuff and to get over and done with
  * that's stuff that, for better or worse, I've started doing
2. `+DUE` tasks (anything non-project related that's urgent can be added quickly) -- definition of 'due' depends on `rc.due` (default 7 days)
  * that's stuff that needs to get done
3. tasks that are `+SCHEDULED` *and* `+READY` (that includes all possibly not-even-remotely-due sprint-tasks)
  * that's stuff that I've decided I *want* to get done

The `:project` report shows all tasks for a project (including completed, at the bottom). incomplete tasks can be in one of 3 states:
1. new tasks are in an unscheduled state, and can be individually turned into 'someday' tasks by pressing `w` or `s` (marking them as `wait:someday`)
2. ~~pressing `x` bulk-marks all non-someday tasks of the project as part of the next `+sprint`~~
3. pressing `X` asks and bulk-sets `scheduled:now` *and* a `due:` date for all of the current project's `+next` tasks. this marks the start of a sprint from now until its due date, which is what will actually make them show up in the `:next` report!

### keybindings

#### important default bindings

* `a` add task
* `b` begin task (set `start:now` which makes the task `+ACTIVE`)
* `d` mark task done (sets the status to completed which makes it `+COMPLETED` with `end:now`)
* `D` delete task (sets the status to deleted, also adds an `end:now`)
* `m` modify task
* `u` undo last action
* `Q` quit

#### custom bindings

bindings with ** at the beginning are sensitive to the currently focussed task's project

##### report navigation

* `q` open `:next` report
* `I` open `:ideas` report (shows *all* `+idea` tasks)
* `K` open `:goals` report (shows *all* `+goal` tasks)
* ** `<Enter>` open `:project` report of the currently focussed task's project
* ** `S` open the sprint view of a project, i.e. its `+PENDING +SCHEDULED` `:project` report of the currently focussed task's project
* `p` call `:project project:` and wait for manual entry of project to filter
* `F` apply extra/custom filter
* `e` inspect/view current task

##### task adding

* `i` add new task with tag `+idea`
* `k` add new task with tag `+goal`
* `W` add a new *weekly* task (`due:eow +weekly `)
* ** `A` add a new task with the same `project:` as the currently focussed task

##### task manipulation/editing

* `Space` set the task `+next`
* `w`, `s` set task *waiting*/*someday* (`wait:someday`)
* `n` make task 'neutral': unset `due:`, `scheduled:` and `wait:` dates
* `f` set task `+frog`
* `o` add simple anotation
* `O` add multi-line (onenote) anotation
* `t` set/modify tags
* `M` modify *all* in view

##### sprint/timing pipeline

* ~~** `x` add `+sprint` to all non-scheduled/waiting tasks of the current project (needs confirmation)~~
* ** `X` bulk-set `scheduled:now` as well as a `due:` date to all `+next` tasks of the current project (needs due date and confirmation)
* `c` sets `start:now` and starts a Clockify time entry via `clockify-cli in {TASK_DESCRIPTION}`
* `C` sets `start:now` AND starts a Clockify time entry with the current task's description *and project* (can fail)

<!-- previous:
`clockify-cli out`
-->

</s>

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
