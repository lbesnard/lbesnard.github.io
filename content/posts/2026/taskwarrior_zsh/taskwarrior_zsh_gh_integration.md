+++
date = '2026-02-11T15:15:43+11:00'
draft = false
title = 'Taskwarrior - Hack your TODO list on the terminal'
+++

# 🚀 Taskwarrior + Zsh + FZF + GitHub: A Terminal-First Workflow

## Why I Built This

My terminal is my main working tool. I use extensively ZSH+TMUX, and open a new panel all the time.

The goal of this setup is simple:

- See my **tasks for today every time I open a terminal/panel**
- Import GitHub issues directly into Taskwarrior using the latest GitHub CLI features.
- Force a morning check-in ritual using a custom Python TUI.
- Use "Life Hacks" over "Discipline" to stop hyperfocusing on fun tasks while avoiding important ones.

I’m very good at avoiding imposed structure. It's been a life long journey.
My TODO list must be in in my face, otherwise, I'll hyperfocus 200% on one fun task and succeed beautifully at avoiding what I don't want to do.

My Current worfklow is now:

- Open terminal first time of the day -> triggers automatically [task-tui](https://github.com/lbesnard/task-tui) to review and add new tasks easily so it's not a burden
- Every time a new terminal or panel is opened, i see my current tasks
  ![Example of tasks list when opening a new terminal](../tasklist.png)

---

# 1. The Philosophy

This setup does three things:

1. **Morning checkpoint (automatic)**
   - When opening the first new terminal session/panel:
     - Review overdue and due-today tasks
     - Close, postpone, or reschedule them
     - Add new tasks intentionally

2. **GitHub → Taskwarrior bridge**
   - Select sprint/iteration issues
   - Import them to Taskwarrior as actionable daily tasks

3. **Fast, keyboard-driven management**
   - Everything via `fzf` and terminal
   - No browser dependency

---

# 2. Requirements

You need:

- [Taskwarrior](https://taskwarrior.org/)
- [task-tui](https://github.com/lbesnard/task-tui)
- [fzf](https://github.com/junegunn/fzf)
- [jq](https://github.com/jqlang/jq)
- [GitHub CLI (`gh`)](https://github.com/cli/cli) – <https://github.com/cli/cli>
- [Zsh](https://www.zsh.org/)/zsh/>
- [tmux](https://github.com/tmux/tmux) (required for auto daily review)

# 3. Task-TUI: The Review Engine

[task-tui](https://github.com/lbesnard/task-tui) is a Terminal User Interface (TUI) for Taskwarrior, built with the Python Textual framework. It provides a seamless, keyboard-driven workflow with live updates, fuzzy searching, and automatic syncing.
p
![Example of task-tui](../tasktui.png)

It replaces complex `task` CLI sequences with fast, discoverable key-driven actions.

### Why It Changes the Game

- **Live Preview & Interaction**: Navigate with Vim-style keys (`j` / `k`), and the detail panel updates instantly.
- **Dynamic Project Colours**: Each project is automatically assigned one of 32+ unique colours for instant visual grouping.
- **Urgency Alerts**: Tasks with urgency > 20 are highlighted in bold red so critical items stand out.
- **Fuzzy Search Everywhere**:
  - `/` to search all pending tasks
  - `Ctrl+F` to search while editing dependencies
- **Dependency Explorer**: Press `v` to view all dependencies in a modal and jump directly to them.
- **Quick Context Actions**:
  - `t` for Quick Due Dates (Today, Tomorrow, End of Week/Month)
  - `p` for Quick Priority changes
- **Batch Operations**: Use `space` to multi-select tasks.
- **Auto-Sync**: Automatically runs `task sync` on startup and exit.

### Core Daily Shortcuts

- `i` → Modify selected task
- `n` → Create new task
- `d` → Mark as Done
- `s` → Start/Stop task
- `x` → Save changes
- `u` → Undo last action
- `q` → Quit and Sync

Task-TUI reads directly from your `.taskrc`. No additional configuration is required. If a `taskserver` is configured, synchronisation happens automatically when the application closes.

Instead of memorising complex `task modify` commands, everything becomes fast, visual, and muscle-memory driven.

---

# 4. GitHub → Taskwarrior Integration

Thanks to a recent [feature request](https://github.com/cli/cli/pull/12696) and the [v2.87.0](https://github.com/cli/cli/releases/tag/v2.87.0) release of the GitHub CLI, we can now run complex queries on Project boards directly.

I use this to fetch my current sprint items and import them into Taskwarrior with `CTRL-T`.

![Example of gh integration](../gh_integration.png)

It:

- Fetches project items
- Filters by assigned user
- Excludes Done/Closed
- Lets you:
  - `ENTER` → open in browser
  - `CTRL-T` → import into Taskwarrior

Add this to your `~/.config/gh/config.yml` and modify the `ORG```` and`PROJECT_NUMBER``` values:

```yaml
project-sprint-taskwarrior: |-
  ! (
    # GH_PATH="/home/lbesnard/github_repo/dotfiles/bin/gh"
    GH_PATH="gh"
    ORG="aodn"
    PROJECT_NUMBER=72

    raw_output=$($GH_PATH project item-list $PROJECT_NUMBER --owner "$ORG" --format json \
      --query "assignee:$GIT_USER iteration:@current -status:Done -status:Closed" | \
    jq -r '
      def color_prio(p):
        if p == "High" then "\u001b[31m" + p + "\u001b[0m"
        elif p == "Medium" then "\u001b[33m" + p + "\u001b[0m"
        elif p == "Low" then "\u001b[32m" + p + "\u001b[0m"
        else "\u001b[90mNone\u001b[0m" end;
      .items[] | 
      (.iteration.title // "No-Iteration") as $iter |
      (.priority // "None") as $prio |
      (.project // "No-Project") as $pjt |
      "\($iter)\t\(color_prio($prio))\t\u001b[36m\($pjt)\u001b[0m\t\(.content.number // "DRAFT")\t\(.content.title)\t\(.content.repository // "none")"
    ' | \
    sort -rV | \
    fzf --ansi --reverse --multi \
        --header "TAB: Select | ENTER: Open | CTRL-T: Taskwarrior" \
        --expect=ctrl-t \
        --delimiter '\t' \
        --with-nth 1,2,3,4,5 \
        --preview-window 'right:60%:wrap' \
        --preview "$GH_PATH issue view {4} -R {6}")

    if [ -z "$raw_output" ]; then exit 0; fi

    key=$(echo "$raw_output" | head -n 1)
    
    echo "$raw_output" | tail -n +2 | perl -pe 's/\e\[[0-9;]*m//g' | while IFS=' ' read -r iter prio pjt num title repo; do
      if [ -z "$num" ] || [ "$num" = "DRAFT" ] || [ "$repo" = "none" ]; then
        continue
      fi

      if [ "$key" = "ctrl-t" ]; then
        tw_prio=$(echo "$prio" | cut -c1 | tr '[:lower:]' '[:upper:]')
        case "$tw_prio" in
          H|M|L) ;;
          *) tw_prio="" ;;
        esac

        echo "📥 Importing #$num: $title"
        url=$($GH_PATH issue view "$num" -R "$repo" --json url --jq .url)
        task add "$title ($url)" project:"$pjt" priority:"$tw_prio" due:today +today +github
      else
        echo "🌐 Opening #$num in browser..."
        $GH_PATH issue view "$num" -R "$repo" --web
      fi
    done
  )
```

Now you can run:

```
gh project-sprint-taskwarrior
```

And import sprint tasks directly into your day with `CTRL-T`.

---

# 5. Automatic Morning Check-in (Zsh + TMUX)

This runs **once per day** when opening a new tmux terminal.

It:

- Reviews pending tasks
- Lets you complete / postpone / delete
- Prompts you to add new tasks
- Shows today's plan

Add this to your `.zshrc`:

```bash
task-today() {
  task rc.verbose=nothing status:pending due:today export | jq -r '
    def color_prio(p):
      if p == "H" then "\u001b[31mHigh\u001b[0m"
      elif p == "M" then "\u001b[33mMed \u001b[0m"
      elif p == "L" then "\u001b[32mLow \u001b[0m"
      else "\u001b[90mNone\u001b[0m" end;
    def pad(len): tostring | . + " " * (len - length);
    if length == 0 then "No tasks for today! " else
      (map(.project // "none") | unique) as $unique_projects |
      [36, 35, 34, 96, 95, 94, 33, 32] as $palette |
      (reduce range(0; $unique_projects | length) as $i ({};
        . + {($unique_projects[$i]): $palette[$i % ($palette | length)]}
      )) as $color_map |
      .[] | (.id | tostring | pad(3)) as $id |
      (.priority // " ") as $raw_prio |
      (.project // "none") as $pjt_raw |
      ($pjt_raw | pad(12)) as $pjt_pad |
      ($color_map[$pjt_raw] // 37) as $code |
      "\u001b[90m\($id)\u001b[0m  \(color_prio($raw_prio))  \u001b[\($code)m\($pjt_pad)\u001b[0m  \(.description)"
    end
  '
}

if command -v task > /dev/null && [[ -n "$TMUX" ]]; then
    LAST_PROMPT_FILE="$HOME/.task_last_prompt"
    TODAY=$(date +%Y-%m-%d)

    if [[ "$(< $LAST_PROMPT_FILE 2>/dev/null)" != "$TODAY" ]]; then
        # echo -e "\e[38;5;81m--- 🔍 Reviewing Overdue & Today's Tasks ---\e[0m"

        export PATH="/home/$USER/miniforge3/bin:$PATH"
        if command -v task-tui > /dev/null; then
          task-tui
        else;
           echo "run python3 -m pip install git+https://github.com/lbesnard/task-tui.git"
        fi

        echo \n
        echo -e "\e[38;5;214m--- 🗓️  Reminder: Fill yesterday's timesheet! ---\e[0m"
          echo "$TODAY" > "$LAST_PROMPT_FILE"
    fi
    echo -e "\e[38;5;81m📅 TODAY'S PLAN\e[0m"
    task-today
    echo ""
fi
```

---

# 6. Core Aliases (Daily Use)

Add some of these to your `.zshrc`:

```bash
## taskwarrior
# Short alias for general use
alias t='task'

# Interactive "Task Done" with Project Label
td() {
    local task_id=$(task status:pending export | jq -r '.[] | "\(.id) [\(.project // "no project")] \(.description)"' | \
        fzf --height 40% --reverse --header "Select task to complete" --preview 'task {1} info' | awk '{print $1}')

    if [[ -n "$task_id" ]]; then
        task "$task_id" done
    fi
}

# Quick view of today's specific deadlines
alias tt='task due:today list'

# Work Tomorrow
alias twt='task add pro:Work due:tomorrow'
# Personal Tomorrow
alias tpt='task add pro:Personal due:tomorrow'

# Work Backlog
alias twb='task add pro:Work'
# Personal Backlog
alias tpb='task add pro:Personal'

alias tw='task add pro:Work due:today'
alias tp='task add pro:Personal due:today'

# Shows everything due in the next 48 hours
alias tnext='task due.before:today+2d list'

# Triage with Project Labels
t2t() {
    local task_ids=$(task status:pending export | jq -r '.[] | "\(.id) [\(.project // "no project")] \(.description)"' | \
        fzf --multi \
            --reverse \
            --header "TAB to select multiple / ENTER to set to Today" \
            --preview 'task {1} info' \
            --height 40% | awk '{print $1}')

    if [[ -n "$task_ids" ]]; then
        echo "$task_ids" | while read -r id; do
            task "$id" modify due:today
        done
        echo "✅ Selected tasks moved to Today."
        task today
    else
        echo "No tasks selected."
    fi
}
tnot() {
    # 1. Get IDs, Project, and Description for today's pending tasks
    local selected=$(task due:today status:pending export | \
        jq -r '.[] | "\(.id) [\(.project // "none")] \(.description)"' | \
        fzf -m --reverse --header "Remove from Today (Clear Due Date)" --height 40%)

    # 2. Extract the IDs
    local ids=$(echo "$selected" | awk '{print $1}')

    # 3. If IDs exist, remove the due date
    if [[ -n "$ids" ]]; then
        local id_list=$(echo $ids | tr '\n' ' ')
        # Setting due: with nothing after it removes the date in Taskwarrior
        task $id_list modify due:
        echo "\e[38;5;208mTasks moved back to backlog.\e[0m"
    fi
}

alias thist='task status:completed end.after:today-1wk list'

# Purge/Delete
tpurge() {
    # 1. Fetch data using UUID (column 1) but hide it from view using --with-nth
    local gen_list='task status:pending export | jq -r ".[] | \"\(.uuid) [\(.project // \"none\")] \(.description) \(if .annotations then \" | NOTES: \" + ([.annotations[] | .description] | join(\" | \")) else \"\" end)\""'

    local selected=$(eval $gen_list | fzf --multi --ansi --reverse \
        --header "TRASH CAN: TAB to select multiple | ENTER to Delete" \
        --with-nth=2.. \
        --preview 'task {1} info' \
        --height 70%)

    if [[ -n "$selected" ]]; then
        # Extract UUIDs and delete them all in ONE command to avoid ID-shifting and prompts
        local uuids=$(echo "$selected" | awk '{print $1}' | tr '\n' ' ')

        # rc.confirmation=no bypasses the "Are you sure?" prompt
        task $uuids delete rc.confirmation=no

        # Optional: Run the actual system purge to clean the database immediately
        task purge rc.confirmation=no
        echo "✅ Selected tasks have been permanently deleted."
    else
        echo "Nothing deleted."
    fi
}

# Select a task and set a due time for today
ttime() {
    local task_id=$(task status:pending export | jq -r '.[] | "\(.id) [\(.project)] \(.description)"' | fzf --reverse --header "Set time for today (HH:MM)")
    if [[ -n "$task_id" ]]; then
        echo -n "Enter time (e.g., 14:00): "
        read task_time
        task ${task_id%% *} modify due:todayT$task_time
    fi
}


# Wait until tomorrow (or a date) to see this task again
alias twait='task $(task status:pending export | jq -r ".[] | \"\(.id) \(.description)\"" | fzf --reverse --header "Wait until when?") modify wait:tomorrow'

tann() {
    local task_id=$(task status:pending export | jq -r '.[] | "\(.id) \(.description)"' | fzf --reverse --header "Add note/annotation")
    if [[ -n "$task_id" ]]; then
        echo -n "Annotation: "
        read note
        task ${task_id%% *} annotate "$note"
    fi
}

alias twork='task project:Work list'
alias tpers='task project:Personal list'

# Interactive "Task Edit" with fzf
te() {
    # Select task, then open the full metadata in your $EDITOR (nvim/vim)
    local task_id=$(task status:pending export | jq -r '.[] | "\(.id) [\(.project // "no project")] \(.description)"' | \
        fzf --reverse --header "Select task to EDIT" --preview 'task {1} info' | awk '{print $1}')

    if [[ -n "$task_id" ]]; then
        task "$task_id" edit
    fi
}

ttd() {
    # 1. Get IDs of tasks due today
    # 2. Use fzf to select (Tab to select multiple)
    # 3. Extract the ID and mark as done
    local ids=$(task due:today status:pending export | \
                jq -r '.[] | "\(.id) \(.description)"' | \
                fzf -m --header "Select tasks to complete (Tab to multi-select)" --height 40% | \
                awk '{print $1}')

    if [[ -n "$ids" ]]; then
        local id_list=$(echo $ids | tr '\n' ' ')
        task $id_list done
    fi
}

ttl() {
    # 1. Filter strictly for pending tasks due today
    local gen_list='task due:today status:pending export | jq -r ".[] | \"\(.id) [\(.project // \"none\")] \(.description)\""'

    # 2. Open FZF with full management capabilities
    eval $gen_list | fzf \
        --reverse --multi \
        --height 80% \
        --header "TAB: Select | ENTER: Info | CTRL-D: Done | CTRL-P: Postpone | CTRL-E: Edit" \
        --preview 'task {1} info' \
        --preview-window=right:65%:wrap \
        --bind "ctrl-d:execute(task {1} done)+reload($gen_list)" \
        --bind "ctrl-p:execute(task {1} modify due:tomorrow)+reload($gen_list)" \
        --bind "ctrl-e:execute(task {1} edit)+reload($gen_list)" \
        --bind "enter:execute(task {1} info | less -R)"
}

```

---

# 6. TLDR; Commands

## Interactive Commands (FZF Powered)

| Command  | Purpose                     |
| -------- | --------------------------- |
| `td`     | Select and complete a task  |
| `t2t`    | Move backlog tasks to Today |
| `ttime`  | Set a specific hour         |
| `te`     | Edit full task metadata     |
| `tann`   | Add annotation              |
| `tnot`   | Remove from Today           |
| `ttd`    | Complete today's tasks      |
| `ttl`    | Interactive Today browser   |
| `tpurge` | Bulk delete tasks           |

Everything is keyboard-driven.

## Adding tasks

| Command      | Action                  | Example                      |
| :----------- | :---------------------- | :--------------------------- |
| `tw <desc>`  | **Work** (Today)        | `tw Fix login bug`           |
| `tp <desc>`  | **Personal** (Today)    | `tp Buy milk`                |
| `twt <desc>` | **Work** (Tomorrow)     | `twt Prep meeting notes`     |
| `tpt <desc>` | **Personal** (Tomorrow) | `tpt Go to the gym`          |
| `twb <desc>` | **Work** (Backlog)      | `twb Research new framework` |
| `tpb <desc>` | **Personal** (Backlog)  | `tpb Plan summer trip`       |

---

# 8. Taskwarrior Configuration

Minimal `.taskrc` additions:

```
report.today.description=Today grouped by Project
report.today.columns=id,project,description
report.today.labels=ID,Project,Description
report.today.sort=project+,urgency-
report.today.filter=status:pending and (due:today or +today)

color.project.Work=color81
color.project.Personal=color208
color.due.today=bold color196

dateformat=YMDTHN
due=Y-M-D H:N
date.iso=YMDTHN
```

---

# Conclusion

The terminal becomes:

- the planner
- the reviewer
- the accountability system
