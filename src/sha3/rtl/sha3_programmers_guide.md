
# Programmer's Guide

## Initialization

The software can update the SHA3 configurations only when the IP is in the idle state.
The software should check [`STATUS.sha3_idle`](sha3_reg.md#status) before updating the configurations.
The update request to [`CFG_SHADOWED`](sha3_reg.md#cfg_shadowed) will be discarded if [`STATUS.sha3_idle`](sha3_reg.md#status) is not 1.
1. The software must program [`CFG_SHADOWED.msg_endianness`](sha3_reg.md#cfg_shadowed) and [`CFG_SHADOWED.state_endianness`](sha3_reg.md#cfg_shadowed) at the initialization stage. These determine the byte order of incoming messages (msg_endianness) and the Keccak state output (state_endianness).
2. The software must configure [`CFG_SHADOWED.mode`](sha3_reg.md#cfg_shadowed) and [`CFG_SHADOWED.kstrength`](sha3_reg.md#cfg_shadowed).
Currently, SHA3 engine only supports SHA3-224, SHA3-256, SHA3-384, SHA3-512, SHAKE-128, SHAKE-256.
Please notices that[`CFG_SHADOWED`](sha3_reg.md#cfg_shadowed) is a shadowed register.
    a. Two subsequent write operation are required to change its content,
    b. If the two write operations try to set a different value, a recoverable alert is triggered (See [`STATUS.ALERT_RECOV_CTRL_UPDATE_ERR`](sha3_reg.md#status)).
    c. A read operation clears the internal phase tracking.
    d. If storage error(~staged_reg!=committed_reg) happen, it will trigger fatal fault alert";


## Software Initiated SHA3 process

This section describes the expected software process to run the SHA3 HWIP.

### Kick-off SHA3 engine
After configuring, the software notifies the SHA3 engine to accept incoming messages by issuing Start command into [`CMD`](sha3_reg.md#cmd) .
If Start command is not issued, the incoming message written to [`MSG_FIFO`](sha3_reg.md#msg_fifo) is discarded.

### Notify SHA3 engine about all message sent done
After the software pushes all messages, it issues Process command to [`CMD`](sha3_reg.md#cmd) for SHA3 engine to complete the sponge absorbing process.
SHA3 hashing engine pads the incoming message as defined in the SHA3 specification.

### Get digest from SHA3 engine
After the SHA3 engine completes the sponge absorbing step, it generates `sha3_done` interrupt([`notif_internal_intr_r.notif_cmd_done_sts`](sha3_reg.md#mnotif_internal_intr_r)).
Or the software can poll the [`STATUS.squeeze`](sha3_reg.md#status) bit until it becomes 1.

In this stage, 
#### SHA3 case
software can read the digest value from [`STATE`](sha3_reg.md#state).
#### SHKKE case
It is same with SHA3 case, if the desire digest length is not greater than Keccak rate.
If the desired digest length is greater than the Keccak rate, after the software reads the current available Keccak state, the software need to issue an Run command for the Keccak round logic to run another full round.
At this stage, SHA3 engine will not raise any interrupts when the Keccak round completes the software initiated manual run.
The software should check [`STATUS.squeeze`](sha3_reg.md#status) register field for the readiness of [`STATE`](sha3_reg.md#state) value after issuing a Run command.

## Software release SHA3

After the software reads all the digest values, it issues Done command to [`CMD`](sha3_reg.md#cmd) register to clear the internal states.
Done command clears the Keccak state, FSM in SHA3, and a few internal variables.
Software programmed values won't be reset.


## Endianness

This SHA3 HWIP operates in little-endian.
Internal SHA3 hashing engine receives in 64-bit granularity.
The data written to SHA3 is assumed to be little endian.

The software may write/read the data in big-endian order if [`CFG_SHADOWED.msg_endianness`](sha3_reg.md#cfg_shadowed) or [`CFG_SHADOWED.state_endianness`](sha3_reg.md#cfg_shadowed) is set.
If the endianness bit is 1, the data is assumed to be big-endian.
So, the internal logic byte-swap the data.
For example, when the software writes `0xDEADBEEF` with endianness as 1, the logic converts it to `0xEFBEADDE` then writes into MSG_FIFO.



## Error Handling

When the SHA3 HW IP encounters an error, it raises the `SHA3_err` IRQ([`notif_internal_intr_r.notif_cmd_done_sts`](sha3_reg.md#mnotif_internal_intr_r)).
SW can then read the `ERR_CODE` CSR to obtain more information about the error.
Having handled that IRQ, SW is expected to clear the `error0_sts` bit in the `error_internal_intr_r` CSR before exiting the ISR.
When SW has handled the error condition, it is expected to set the `err_processed` bit in the `CMD` CSR.
The internal SHA3 engine then flushes its FIFOs and state, which may take a few cycles.
The SHA3 HW IP is ready for operation again as soon as the `sha3_idle` bit in the `STATUS` CSR is set; SW must not change the configuration of or send commands to the SHA3 HW IP before that.
If the error occurred while the SHA3 HW IP was being used from SW (i.e., not via an HW application interface), the `sha3_done` IRQ is raised when the SH3 HW IP is ready again.


## SHA3 context switching

This version of sSHA3 HWIP _does not_ support the software context switching.
A context switching scheme would allow software to save the current hashing engine state and initiate a new high priority hashing operation.
It could restore the previous hashing state later and continue the operation.

## Device Interface Functions (DIFs)

- [Device Interface Functions](../../../../sw/device/lib/dif/dif_kmac.h)
