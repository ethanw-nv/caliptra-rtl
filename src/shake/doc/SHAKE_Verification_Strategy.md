# SHAKE Verification Strategy

## Goals
- Verify all SHAKE IP features by running dynamic simulations with a SV/UVM based testbench
- Develop and run all tests based on the [testplan](#testplan) below towards closing code and functional coverage on the IP and all of its sub-modules(except ...)

## Current Status
- Testbench and RTL are under development
- Initial architecture and components are being designed

## Design features
- Support Algorithms (inherit from Open Titan):
   - SHAKE-128
   - SHAKE-256
   - SHA3-256
   - SHA3-512
- Support MMIO access to the SHAKE accelerator through AHB-lite
   - Support AHB-lite to Open Titan TL-UL protocol conversion
- Support event notification, e.g., task completion, through interrupt
- Support error reporting through interrupt

## Testbench Architecture

### Top level testbench
Top level testbench is located at `xxx`. It instantiates the SHAKE DUT module `xxx`.
In addition, it instantiates the following interfaces, connects them to the DUT and sets their handle into `uvm_config_db`:
- Clock and reset interface
- AHB-lite intrface
- SHAKE IOs
- Interrupts

### AHB Agent
- Implements AHB protocol interface
- Handles read/write transactions
- Monitors bus protocol compliance
- Provides transaction-level interface for sequences

### UVM RAL Model
- Register map for SHAKE control and status registers
- Memory map for input/output data buffers
- Configuration registers for algorithm selection
- Status registers for operation completion and errors

### Reference Models
- Collects all input configurations from RAL 
- Utilizes a DPI-C to calculate expected results 
- Send expected results to scoreboard

### Scoreboard
- Receive expected txn from reference model
- Receive actual txn from monitor
- Compare actual txn with expected txn

## Stimulus Strategy
### Test Sequences
- Input/Output data from NIST vectors
- Random input data generation
- Edge cases and corner cases
- Various message lengths
- Different output lengths for SHAKE
- Negative test for error handling

## Functional Coverage
- Input message length coverage
- Output length coverage (for SHAKE)
- Algorithm selection coverage
- Error condition coverage

## Assertions
- Protocol compliance assertions

## Building and Running Tests
- Makefile-based for buildint and running test
- Regression test suite
- Coverage collection and analysis
- Waveform generation and viewing

## Testplan
1. Phase 1: Basic Functionality
   - Basic algorithm verification
   - NIST vetors test cases
   - Random configurations test cases

2. Phase 2: Advanced Features
   - AHB Bus protocal compliance(assume do NOT need to be signed off)
   - Registers attribute
   - Stress test
   - Corner test

3. Phase 3: Negative Case Test
   - Error handling
   - Interrupt handling

4. Phase 4: Integration Test
   - System-level testing