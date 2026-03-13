# Anthropic Academy — Hook Challenge

![repo-cover](repo-cover.png)

A challenge project exploring **Claude Code Hooks** — deterministic wrappers that run before or after Claude uses a tool, letting you enforce behavior that can't be lost to prompt drift.

---

## What Are Hooks?

Hooks can be used before or after claude code does something, or to block an action.
This can be used to determinately drive behavior instead of relying on steering it to do so with instructions that could be lost.

Could be used to:
- Run tests automaticallu after a file is changed
- Block depreacated fucntion useage
- Check for TODO comments in code tthat claude writes and and add them to a log file

Hooks are defined in `settings.local.json` in `/.claude`.
Hooks happen at the wrapper end just before it uses a tool or after.
Hooks can be written by hand or by using `/hooks` command in claude code.

---

## Hook Types

### PreToolUse

Runs just before Claude uses a tool. Can **block** the tool call and send an error message back to Claude.

```json
"PreToolUse": [
  {
    "matcher": "Read",
    "hooks": [
      {
        "type": "command",
        "command": "node /home/hooks/read_hook.ts"
      }
    ]
  }
]
```

### PostToolUse

Runs after Claude uses a tool. Can provide **additional feedback** to Claude after the tool was ran.

```json
"PostToolUse": [
  {
    "matcher": "Write|Edit|MultiEdit",
    "hooks": [
      {
        "type": "command",
        "command": "node /home/hooks/edit_hook.ts"
      }
    ]
  }
]
```

- The `"matcher":` string is used to find cases of the tool used
- The `"command":` is the command to run
- Tools are seperated witht he pipe symble `|`

---

## How It Works

Claude feeds in tool call data as JSON to the hook command via stdin:

```json
{
  "session_id": "2d6a1e4d-6...",
  "transcript_path": "/Users/sg/...",
  "hook_event_name": "PreToolUse",
  "tool_name": "Read",
  "tool_input": {
    "file_path": "/code/queries/.env"
  }
}
```

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | All is well |
| `2` | Tool blocked (PreToolUse only) — stderr is sent back to Claude as feedback |

---

## Building a Hook

In buildiing a hook you need to:

1. Decide on a PreToolUse or PostToolUse hook
2. Dertermine which type of toll calls you want to watcjh for
3. Wriate a command tha twi;;; recive the too; ca;;
4. If needed cammand shou;d frovide feedback to claude via stderr and the appropriate exit code

---

## Challenge: Block `.env` from Being Read

The challenge is to write a hook that prevents Claude from reading `.env` files.

### Settings Config

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node ./hooks/read_hooks.js"
          }
        ]
      }
    ]
  }
}
```

We define a `PreToolUse` with `"matcher": "Read|Grep"` to catch all read and grep commands. We would than define a command with `"type": "command"` followed with `"command": "{actual code to run}"` — we would need to make our own script to actually have determanistic behavior of checking if the file is `.env`.

### Hook Script

```js
async function main() {
  const chunks = [];
  for await (const chunk of process.stdin) {
    chunks.push(chunk);
  }
  const toolArgs = JSON.parse(Buffer.concat(chunks).toString());

  // readPath is the path to the file that Claude is trying to read
  const readPath =
    toolArgs.tool_input?.file_path || toolArgs.tool_input?.path || "";

  // Ensure Claude isn't trying to read the .env file
  if (readPath.includes(".env")) {
    console.error(
      ".env contains sensitive information, you cannot access or read it.",
    );
    process.exit(2);
  }
}

main();
```

### Result

After renaming the example settings file and relaunching claude it can no longer read this file:

![hook-demo](hook-demo.png)

---

## File Structure

```
.
├── hooks/
│   └── read_hooks.js       # Hook script to block .env access
└── .claude/
    └── settings.local.json # Hook configuration
```
