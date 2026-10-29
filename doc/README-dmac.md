# DMAC

## Overview

This Software Building Block (SBB) demonstrates basic Direct Memory Access Controller (DMAC) functionality on Renesas RA MCUs using Renesas FSP. The reference application creates three DMAC transfer use cases that show peripheral-triggered and software-triggered data movement without using the CPU to perform each individual data copy.

This document covers only these reference boards:

- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

The three demonstrated transfers are:

1. An AGT-triggered transfer from a source array to the I/O port control register, toggling an on-board LED 60 times.
2. A software-triggered 32-bit transfer from the GPT counter register to a destination array, with transferred values printed to the terminal.
3. A software-triggered repeat-mode transfer from a predefined source array to the I/O port control register, changing the LED state.

## Hardware requirements

- 1 x FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1
- 1 x appropriate USB cable for programming, debugging, and the serial terminal connection

Connect the board USB debug port to the host PC.

### FPB-RA4E2

No additional board-specific hardware connection is identified for FPB-RA4E2 in the supplied `readme.txt`.

### FPB-RA6E2

No additional board-specific hardware connection is identified for FPB-RA6E2 in the supplied `readme.txt`.

### FPB-RA8E1

Open jumpers **J6** and **J9** to use the on-board debug functionality.

## What use case does this SBB demonstrate?

The SBB demonstrates three different ways DMAC can move data between memory and peripheral registers.

### 1. Peripheral-triggered LED transfer

Selecting menu option `1` runs `transfer_led_blink_operation()`.

The operation:

1. Sets the DMAC source to `g_source_data`.
2. Sets the destination to the board-specific I/O port `PCNTR1` register.
3. Opens and enables `g_transfer_led_blink`.
4. Opens and starts the AGT timer.
5. Uses the AGT-generated activation events to drive the transfer.
6. Waits until the DMAC callback sets `g_is_transfer_complete`.

The source array contains 60 32-bit values, and the supplied documentation describes AGT as generating a 100 ms interrupt that triggers 60 transfers. The visible result is the on-board LED toggling 60 times.

### 2. GPT register-to-memory transfer

Selecting menu option `2` performs a software start of `g_transfer_gpt_value`.

Before the user selects this option, the application configures the DMAC source as the GPT0 counter register:

```c
&R_GPT0->GTCNT
```

and the destination as `g_dest_data`. GPT0 is opened and started and continues running. When option `2` is selected, the DMAC performs the configured transfer and the application prints entries from the destination array.

### 3. Software-triggered I/O port transfer

Selecting menu option `3` software-starts `g_transfer_write_ioport`.

This instance is configured by the SBB to transfer data from `g_source_data` to the board-specific `PCNTR1` register. The source describes this transfer as repeat mode with software activation. Writing the prepared values to the port control register changes the LED state.

## What does the SBB core code do?

The reusable SBB functionality is divided across three source groups.

### DMAC transfer operations

`dmac_transfers.c` and `dmac_transfers.h` implement the three demonstrated operations:

- `transfer_led_blink_operation()`
- `transfer_gpt_timer_operation()`
- `transfer_write_ioport_operation()`

This layer connects application-level operations to the corresponding DMAC configuration, source and destination addresses, timer resources, and board-specific port register.

### DMAC initialization and primitive operations

`transfer_initialise.c` and `transfer_initialise.h` provide the lower-level reusable DMAC support:

- DMAC open and enable
- DMAC close
- Software-start of a transfer
- Source/destination address assignment
- Destination storage for GPT counter values
- Board-specific source data for LED control
- Printing of captured GPT transfer data

### Timer initialization

`timer_initialise.c` and `timer_initialise.h` provide the timer support required by the transfer demonstrations:

- GPT open/start and close
- AGT open/start and close

GPT supplies the changing counter-register data used by the second transfer. AGT supplies activation events for the first transfer.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

The reusable implementation is primarily:

- `dmac_transfers.c`
- `dmac_transfers.h`
- `transfer_initialise.c`
- `transfer_initialise.h`
- `timer_initialise.c`
- `timer_initialise.h`

These files implement DMAC initialization, transfer setup, timer support, transfer operations, and board-specific LED-port handling.

### Demo/reference application code

`dmac_example_app.c` and `dmac_example_app.h` provide the interactive reference application.

The demo layer:

- Initializes the UART console.
- Reads the FSP version.
- Displays the SBB information and menu.
- Initializes the GPT-based and repeat-mode demonstrations.
- Processes menu selections `1`, `2`, and `3`.
- Prints GPT destination data.
- Reports errors and performs cleanup.
- Runs continuously waiting for terminal input.

The demo entry point is:

```c
void dmac_example_app_run(void);
```

It is intended to be called from `hal_entry.c`.

Console interaction and menu handling are demonstration concerns rather than requirements of the underlying DMAC transfer functions.

## Which public functions should I look at first?

### `transfer_led_blink_operation()`

```c
fsp_err_t transfer_led_blink_operation(void);
```

The best starting point for the event-triggered DMAC example. It assigns the array and I/O-port addresses, initializes the DMAC transfer, starts AGT, and waits for the transfer-completion callback.

### `transfer_gpt_timer_operation()`

```c
fsp_err_t transfer_gpt_timer_operation(void);
```

Sets the GPT0 counter register as the source, `g_dest_data` as the destination, initializes the corresponding DMAC instance, and starts GPT0.

### `transfer_write_ioport_operation()`

```c
fsp_err_t transfer_write_ioport_operation(void);
```

Configures the source array and board-specific I/O port register for the repeat-mode, software-activated transfer.

### `dmac_transfer_init()`

```c
fsp_err_t dmac_transfer_init(dmac_instance_ctrl_t * const p_transfer_ctl,
                             transfer_cfg_t const * const p_transfer_cfg,
                             uint8_t transfer_unit);
```

Opens a DMAC instance with `R_DMAC_Open()` and enables its configured activation source with `R_DMAC_Enable()`.

### `dmac_transfer_software_start()`

```c
fsp_err_t dmac_transfer_software_start(transfer_ctrl_t * const p_transfer_ctrl);
```

Starts a configured DMAC instance using `R_DMAC_SoftwareStart()` with `TRANSFER_START_MODE_SINGLE`.

### `set_transfer_dst_src_address()`

```c
void set_transfer_dst_src_address(transfer_cfg_t const * const p_config,
                                  void const * volatile p_src,
                                  void const * volatile p_dest);
```

Assigns the source and destination addresses in the transfer configuration before the DMAC instance is opened.

### `gpt_timer_init()` and `agt_timer_init()`

These functions open and start the generated GPT and AGT instances used by the demonstrations.

## Configuration assumptions for the reference boards

### Three generated DMAC instances

The SBB expects three generated DMAC configurations:

- `g_transfer_led_blink`
- `g_transfer_gpt_value`
- `g_transfer_write_ioport`

The first supports timer-triggered LED control, the second captures GPT counter data, and the third supports a software-triggered repeat-mode I/O-port transfer.

### Transfer data sizes

The SBB declares:

```c
#define SOURCE_DATA_SIZE (60U)
#define DEST_DATA_SIZE   (16U)
```

`g_source_data` therefore contains 60 board-specific 32-bit values, while `g_dest_data` holds 16 32-bit values.

### Board-specific I/O port registers

The SBB writes directly to an I/O port `PCNTR1` register. For the requested reference boards:

| Board | I/O port control register |
|---|---|
| FPB-RA4E2 | `R_PORT2->PCNTR1` |
| FPB-RA6E2 | `R_PORT2->PCNTR1` |
| FPB-RA8E1 | `R_PORT4->PCNTR1` |

The implementation describes `PCNTR1` as a 32-bit register where bits `[15:0]` control pin direction and bits `[31:16]` control output level.

### Board-specific source data

FPB-RA4E2 and FPB-RA6E2 use the same 60-element LED source-data sequence in the supplied code, alternating:

```text
0x00000080
0x00800080
```

FPB-RA8E1 uses a separate sequence alternating:

```text
0x00000010
0x00100010
```

These values are coupled to the port register and LED configuration used by the reference boards.

### GPT source register

The GPT-transfer demonstration explicitly uses:

```c
R_GPT0->GTCNT
```

as the source register.

### AGT timing

The supplied `readme.txt` describes the AGT source as generating an interrupt every **100 ms** for the LED-transfer demonstration. The SBB performs 60 transfers, and the reference application tells the user to wait 6 seconds while that operation is running.

## What must I change for another configuration?

### Board and LED port

The LED transfer is coupled directly to a board-specific `PCNTR1` register and to values in `g_source_data`. Changing the LED pin or port therefore requires the destination register and corresponding source data to be reviewed together.

For the three documented boards, the existing implementation explicitly selects Port 2 for FPB-RA4E2/FPB-RA6E2 and Port 4 for FPB-RA8E1.

### DMAC source and destination

The SBB assigns source and destination pointers before opening the transfer interface because the code states that opening the transfer will fail if those addresses are not set first.

If the transferred resource changes, update the source/destination selection to the addresses required by that use case and ensure that the generated DMAC configuration matches it.

### Activation source and transfer mode

The demonstrations use both peripheral activation and software activation. The transfer mode and activation source are part of the generated DMAC configuration. They must remain consistent with the intended transfer sequence.

### Timers and clocks

The source uses generated GPT (`g_gpt_ctrl`, `g_gpt_cfg`) and AGT (`g_agt_ctrl`, `g_agt_cfg`) configurations. The supplied files do not state the timer clock source, divider, or clock-tree configuration for FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1. If the target's clock configuration changes, validate the generated timer configuration and the required activation timing.

### Timing

The documented AGT-triggered example uses a 100 ms trigger and 60 transfers. If the application requires a different transfer rate or count, the AGT and DMAC transfer configuration must be changed consistently.

### Memory

The supplied implementation has a 60-element 32-bit source array and a 16-element 32-bit destination array. The source files do not provide an overall RAM/ROM footprint for the SBB.

### Task priorities

The supplied implementation does not define an RTOS or task priorities. `transfer_led_blink_operation()` waits in a foreground busy loop until the transfer callback changes `g_is_transfer_complete`. An RTOS-based integration must define its own scheduling requirements.

### Console setup

The UART console is used by the demonstration for menu input and output. It is not used by the DMAC hardware to perform the transfers. If the demo UI is removed, the integrating application can invoke the reusable transfer functions through its own control path.

## Dependencies

### Renesas FSP DMAC driver

The SBB directly uses:

- `R_DMAC_Open()`
- `R_DMAC_Enable()`
- `R_DMAC_SoftwareStart()`
- `R_DMAC_Close()`

### Renesas FSP timers

The demonstrations use:

- `R_AGT_Open()`
- `R_AGT_Start()`
- `R_AGT_Close()`
- `R_GPT_Open()`
- `R_GPT_Start()`
- `R_GPT_Close()`

### Hardware registers and I/O

The implementation accesses:

- `R_GPT0->GTCNT`
- `R_PORT2->PCNTR1` for FPB-RA4E2 and FPB-RA6E2
- `R_PORT4->PCNTR1` for FPB-RA8E1

### Generated FSP configuration

The handwritten SBB source depends on objects exposed through `hal_data.h`, including the three DMAC configurations and the AGT/GPT configurations.

### Console/reference application

The demo uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `fgets()`
- `atoi()`
- `memset()`
- `strcspn()`
- `R_FSP_VersionGet()`

## Serial terminal configuration

The original README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term, it additionally specifies:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## Demonstrated behavior and performance

### Functional behavior demonstrated

The supplied application demonstrates:

- Opening and enabling three DMAC transfer instances.
- Peripheral-triggered DMAC operation using AGT.
- Software-triggered DMAC operation.
- Normal-mode transfer to an I/O port register.
- Repeat-mode transfer to an I/O port register.
- 32-bit transfer from the GPT counter register to memory.
- Direct DMAC writes to an I/O port control register to change an LED state.
- Transfer-completion notification through the DMAC callback.

### Demonstrated timing

For the LED-blink transfer, the README specifies a **100 ms AGT interrupt** and **60 transfers**. The reference application labels that operation as requiring a **6 second** wait.

### Known behavior

The README notes that on starting or resetting a board, the DMAC transfer can sometimes take longer, causing LED blinking to decrease in frequency or pause. It instructs the user to reset the board if this occurs.

### Performance measurements

The supplied files do not provide measured DMAC throughput, bus utilization, CPU utilization, interrupt latency, memory footprint, jitter, or transfer latency. The 100 ms trigger period and 60-transfer example describe the configured demonstration, not a DMAC performance limit.

This SBB should therefore be treated as a functional DMAC reference rather than a performance benchmark.

## File ownership and modification guidance

| File | Classification | Guidance |
|---|---|---|
| `dmac_transfers.c` | Reusable SBB transfer operations | Modify when intentionally changing the demonstrated DMAC source/destination resources or transfer behavior. Board-specific destination-register selection is here. |
| `dmac_transfers.h` | Reusable SBB transfer interface | Modify when changing the public high-level transfer operations. |
| `transfer_initialise.c` | Reusable SBB DMAC support and board presets | Contains DMAC wrappers, buffers, and board-specific LED source data. Modify carefully because the data values are coupled to board I/O configuration. |
| `transfer_initialise.h` | Reusable SBB DMAC interface/constants | Defines transfer IDs, menu operation IDs, buffer sizes, and primitive DMAC functions. |
| `timer_initialise.c` | Reusable timer support | Modify if the SBB's AGT/GPT initialization and cleanup behavior must change. |
| `timer_initialise.h` | Reusable timer interface | Public declarations for the SBB timer helpers. |
| `dmac_example_app.c` | Demo/reference application | Appropriate location for menu flow, terminal interaction, demonstration sequencing, and demo error handling. |
| `dmac_example_app.h` | Demo/reference application interface | Contains the demo entry point and presentation/menu constants. |
| `readme.txt` | Existing example documentation | Source of supported-board, hardware, known-behavior, and terminal information. |

The SBB also depends on generated FSP content through `hal_data.h`. The provided file set does not enumerate those generated files, so individual generated files are not classified here.

## What happens if I modify the SBB and later adopt a newer release?

Local changes to the reusable SBB files can overlap changes delivered by a later SBB release. This is particularly important for this building block because board-specific register selection and board-specific LED source sequences are present in the reusable source.

To reduce upgrade conflicts:

- Keep terminal menus, messages, and other demonstration behavior in `dmac_example_app.*`.
- Prefer representing timer and DMAC configuration in the generated FSP configuration when the configuration belongs there.
- Keep unavoidable board-specific changes to `dmac_transfers.c` and `transfer_initialise.c` focused and documented.
- Keep generated FSP content separate from hand-maintained SBB source.
- When updating the SBB, compare the new reusable files with locally modified versions before carrying local changes forward.
- Revalidate DMAC source/destination addresses, board port-register selection, LED source-data values, AGT activation, GPT register capture, transfer modes, and callback completion after merging an update.

The supplied files do not define an automated source-migration or compatibility mechanism for future SBB versions. Compatibility after an update therefore needs to be validated by the integrating project.

## Recommended reading order

1. **`dmac_transfers.h`** to understand the three high-level SBB operations.
2. **`dmac_transfers.c`** to see how each operation combines DMAC, timers, buffers, and board-specific port registers.
3. **`transfer_initialise.h`** to understand transfer identifiers, data sizes, and the lower-level DMAC API.
4. **`transfer_initialise.c`** to see DMAC initialization, software start, address assignment, destination data, and board-specific source arrays.
5. **`timer_initialise.h`** and **`timer_initialise.c`** to understand the AGT and GPT dependencies.
6. **`dmac_example_app.c`** to see how the three operations are exercised from the terminal menu.
7. **`dmac_example_app.h`** for the demo entry point and menu/information strings.
8. **`readme.txt`** for the original hardware and terminal notes.
