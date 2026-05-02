---
description: List topic clusters or resume one in a new terminal window (uses ~/bin/topic)
argument-hint: [keyword]
---

User invoked `/topic`. Args: `$ARGUMENTS`

Run the following via the Bash tool. Use the literal `topic` command (it's on the user's PATH at `~/bin/topic`).

**If `$ARGUMENTS` is empty:**
1. Run `topic list`.
2. Show the user the cluster table verbatim.
3. Ask which topic they want to resume. Once they pick, run step 2 below with their pick.

**If `$ARGUMENTS` has a keyword:**
1. Run `topic --spawn $ARGUMENTS`.
2. Two possible outcomes:
   - **Spawned**: stdout contains `Opened new terminal window: <title>`. Tell the user a new window opened with that topic loaded. Their current session is unaffected.
   - **Fallback**: stdout contains `Could not spawn new window. Run this manually:` followed by a command. Tell the user the command and offer to `pbcopy` it for them so they can paste and run.
3. If `topic` reports no cluster or session matched, tell the user that and ask what they meant. Do not invent topics.

Do not paraphrase what `topic` returns. Use the verbatim output for the topic list and the launch command.
