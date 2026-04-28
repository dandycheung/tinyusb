---
name: hil
description: Use when running TinyUSB Hardware-in-the-Loop (HIL) tests on physical boards, debugging HIL failures, or copying firmware to the ci.lan test rig. Covers local execution and remote execution over SSH, config selection, and debugging tips.
---

# Hardware-in-the-Loop (HIL) Testing

Run TinyUSB HIL tests against real boards. Two execution modes — **local** (boards attached to this machine) and **remote** (boards attached to `ci.lan`, reached over SSH). Default to **local** unless the user specifies `remote`. Do not auto-detect.

## Prerequisites

- Examples must already be built for the target board(s). See AGENTS.md "Build" section, Option 2 (all examples for a board), which produces `examples/cmake-build-BOARD_NAME/`.
- `-B examples` tells `hil_test.py` that `examples/` is the parent folder containing the per-board build outputs.

## Choosing arguments

Infer from the user's request:

- **Mode:** `local` (default) or `remote`. Only switch to `remote` if the user explicitly says so or names `ci.lan`.
- **Board:** if the user names a specific board, pass `-b BOARD_NAME`. Otherwise run all boards in the config.
- **Pass-through flags:** `-v` (verbose), `-r N` (retry count), etc. — pass through unchanged.

Config file follows from mode:
- **Local** → `local.json`
- **Remote** → `tinyusb.json`

## Local execution

Boards attached to this machine:

```bash
python test/hil/hil_test.py -b BOARD_NAME -B examples local.json $EXTRA_ARGS
# or for all boards in the config:
python test/hil/hil_test.py -B examples local.json $EXTRA_ARGS
```

## Remote execution (ci.lan)

Copy only the minimal files needed (firmware binaries + test script + config), then run remotely:

```bash
REMOTE=ci.lan
REMOTE_DIR=/tmp/tinyusb-hil

# Create remote working directory
ssh $REMOTE "rm -rf $REMOTE_DIR && mkdir -p $REMOTE_DIR/test/hil"

# Copy HIL test script and its dependency
scp test/hil/hil_test.py test/hil/pymtp.py test/hil/tinyusb.json $REMOTE:$REMOTE_DIR/test/hil/

# Copy firmware binaries
# Specific board:
scp -r examples/cmake-build-$BOARD_NAME $REMOTE:$REMOTE_DIR/examples/
# Or all built boards:
# for dir in examples/cmake-build-*/; do scp -r "$dir" $REMOTE:$REMOTE_DIR/examples/; done

# Run the test remotely
ssh $REMOTE "cd $REMOTE_DIR && python3 test/hil/hil_test.py -b $BOARD_NAME -B examples tinyusb.json $EXTRA_ARGS"
```

The remote machine (`ci.lan`) must have:
- Python 3 with `pyserial` installed (`pip install pyserial`)
- Flasher tools: `JLinkExe`, `openocd`, etc. as needed by the board
- USB access to the boards (udev rules configured)

## Timing

HIL runs take 2-5 minutes. Use a timeout of at least 20 minutes (600000 ms). NEVER cancel early.

## Reporting results

After the test completes:
- Show the test output to the user.
- Summarize pass/fail per board.
- On failure, suggest re-running with `-v` for verbose output. If `-v` isn't enough, temporarily add debug prints to `test/hil/hil_test.py` to pinpoint the issue.
