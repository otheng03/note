# Shortcuts

## Editor
- shift command A (or double shift) : Search Action
  - For example, Search Action -> Preview

## Compile
- command F9 : Build project

## Debug
- command F8 : Toggle line breakpoint
- option F10 : When degugging, this jumps to the current execution line

### CLion
#### Enable breakpoints of child process
1. Build, Execution, Deployment > Toolchains -> Check if debugger is GDB
2. Execute Debug
3. Open GDB panel and input the below commands
```
(gdb) set follow-fork-mode child
(gdb) set detach-on-fork off
```

