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
- Import **GitHub issues directly into Taskwarrior**
- Force a **morning check-in ritual**
- Close yesterday properly before starting today

I’m very good at avoiding imposed structure. It's been a life long journey.
My TODO list must be in in my face, otherwise, I'll hyperfocus 200% on one fun task and succeed beautifully at avoiding what I don't want to do.

So instead of relying on discipline, I must use life hacks.

Now:

- Open terminal -> Check on TODO list and focus on what needs to be done.

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
   - Everything via `fzf`
   - No browser dependency
   - No mental friction

---

# 2. Requirements

You need:

- [Taskwarrior](https://taskwarrior.org/) – https://github.com/GothenburgBitFactory/taskwarrior
- [fzf](https://github.com/junegunn/fzf) – https://github.com/junegunn/fzf
- [jq](https://github.com/jqlang/jq) – https://github.com/jqlang/jq
- [GitHub CLI (`gh`)](https://github.com/cli/cli) – https://github.com/cli/cli
- [Zsh](https://www.zsh.org/) – https://sourceforge.net/projects/zsh/
- [tmux](https://github.com/tmux/tmux) (required for auto daily review)

---

# 3. GitHub → Taskwarrior Integration

This is a custom `gh` alias.

It:

- Fetches project items
- Filters by assigned user
- Excludes Done/Closed
- Lets you:
  - `ENTER` → open in browser
  - `CTRL-T` → import into Taskwarrior

Add this to your `~/.config/gh/config.yml` and modify the `ORG```` and `PROJECT_NUMBER``` values:

```yaml
project-sprint-taskwarrior: |-
  ! (
    ORG="aodn"
    PROJECT_NUMBER=72

    output=$(gh project item-list $PROJECT_NUMBER --owner "$ORG" -L 1000 --format json --jq '
      .items[] | 
      select(.assignees // [] | contains(["'"$GIT_USER"'"])) | 
      select(.status != "Done" and .status != "Closed") | 
      ((.iteration.title // .milestone.title) // "No-Iteration") as $iter | 
      select($iter != "No-Iteration") |
      "\($iter)\t\(.content.number // "DRAFT")\t\(.content.title)\t\(.content.repository // "none")"' | \
      sort -rV | \
      fzf --reverse --multi \
          --header "TAB: Select Multiple | ENTER: Open | CTRL-T: Add to Taskwarrior" \
          --expect=ctrl-t \
          --delimiter '\t' \
          --with-nth 1,2,3)

    [ -z "$output" ] && exit 0

    key=$(echo "$output" | head -n 1)
      
    echo "$output" | tail -n +2 | while IFS= read -r line; do
      [ -z "$line" ] && continue

      num=$(echo "$line" | cut -f2)
      title=$(echo "$line" | cut -f3)
      repo=$(echo "$line" | cut -f4)

      [ "$repo" = "none" ] && continue

      if [ "$key" = "ctrl-t" ]; then
        url=$(gh issue view "$num" -R "$repo" --json url --jq .url)
        task add "$title ($url)" project:Work due:today +today +github
      else
        gh issue view "$num" -R "$repo" --web
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

# 4. Automatic Morning Check-in (Zsh + TMUX)

This runs **once per day** when opening a new tmux terminal.

It:

- Reviews pending tasks
- Lets you complete / postpone / delete
- Prompts you to add new tasks
- Shows today's plan

Add this to your `.zshrc`:

```bash
if command -v task > /dev/null && [[ -n "$TMUX" ]]; then
    LAST_PROMPT_FILE="$HOME/.task_last_prompt"
    TODAY=$(date +%Y-%m-%d)

    if [[ "$(< $LAST_PROMPT_FILE 2>/dev/null)" != "$TODAY" ]]; then
        echo "\e[38;5;81m--- 🔍 Reviewing Overdue & Today's Tasks ---\e[0m"

        local REFRESH_CMD="task rc.verbose=nothing status:pending export | jq -r '.[] | \"\(.id) | \(.project // \"none\") | \(.due // \"none\") | \(.description)\"'"

        while true; do
            local fzf_input=$(eval "$REFRESH_CMD")
            [[ -z "$fzf_input" ]] && break

            local selected=$(echo "$fzf_input" | fzf \
                --header "ALT-X: Done | ALT-C: Delete | ALT-P: Postpone | ALT-T: Set Today | ENTER: Finish" \
                --multi --ansi --no-bold \
                --bind "alt-x:execute(echo {+1} | xargs -I{} task {} done)+reload($REFRESH_CMD)" \
                --bind "alt-c:execute(echo {+1} | xargs -I{} task {} delete)+reload($REFRESH_CMD)" \
                --bind "alt-p:execute(echo {+1} | xargs -I{} task {} modify due:)+reload($REFRESH_CMD)" \
                --bind "alt-t:execute(echo {+1} | xargs -I{} task {} modify due:$TODAY)+reload($REFRESH_CMD)" \
                --preview "task {1} info" --preview-window=bottom:3:wrap)

            break
        done

        echo "\n\e[38;5;208mAdd tasks (Empty line to finish):\e[0m"

        while true; do
            echo -n "❯ "
            read -r input
            [[ -z "$input" ]] && break
            task add "$input" due:today
        done

        echo "$TODAY" > "$LAST_PROMPT_FILE"
        task sync
        clear
    fi

    echo "\e[38;5;81m📅 TODAY'S PLAN\e[0m"
    task today
    echo ""
fi
```

Result:

- You cannot start coding without seeing your commitments unless you type `CTRL-C`.
- You cannot ignore unfinished work.
- You cannot “drift”.

---

# 5. Core Aliases (Daily Use)

Add these to your `.zshrc`:

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

# 7. Taskwarrior Configuration

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
