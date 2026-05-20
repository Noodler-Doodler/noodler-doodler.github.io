# 5-Stage DLX Pipeline Processor

!!! abstract "Project Overview"
    **Objective:** Architected and synthesized a custom 32-bit pipelined DLX instruction set architecture featuring hardware-based hazard resolution, branch prediction, and peripheral I/O integration.

    **Hardware:** DE10-Lite FPGA

    **Tech Stack:** `VHDL` `Python` `Assembly` `Questa Simulator` `Signal Tap`

## System Architecture
The core of this project is a custom 5-stage pipeline consisting of Instruction Fetch (IF), Instruction Decode (ID), Execute (EX), Memory (MEM), and Writeback (WB) stages. Unlike standard byte-addressable MIPS architectures, this processor utilizes a word-addressable instruction memory and direct branch addressing rather than pc-relative branching.

Custom I/O instructions and peripherals were added to the ISA to allow user interaction through terminal and benchmarking of the processor performance through custom timing hardware. 


### The 5-Stage Pipeline

![5 stage DLX pipeline](/assets/images/fullpipeline.drawio.png)
/// caption
DLX pipeline block diagram with high level data flow
///

The above block diagram illustrates the complete simplified data path and control logic of the custom 5-stage pipelined processor, segmented into Instruction Fetch (IF), Instruction Decode (ID), Execute (EX), Memory (MEM), and Writeback (WB) stages. The forwarding paths, multiplexors, branch prediction hardware, and peripherals were excluded from the diagram for simplicity. These additional aspects of the architecture are outlined below in the following subsections.

Execution begins in the IF stage, where the Program Counter (PC) utilizes a word-addressable memory architecture—incrementing strictly by 1 rather than 4—while a Branch Target Buffer (BTB) simultaneously predicts control flow to minimize penalty cycles. As the instruction enters the ID stage, operands are pulled from the Register File, and a dedicated Hazard Detection Unit monitors for unavoidable dependencies, issuing stalls when necessary. During the EX stage, the Arithmetic Logic Unit (ALU) computes the results, heavily supported by a Forwarding Unit that actively resolves Read-After-Write (RAW) data hazards by routing data directly from the EX/MEM or MEM/WB pipeline registers back into the ALU multiplexers. Finally, the instruction traverses the MEM stage for any requisite Data Memory access before the WB stage commits the ultimate computed value back to the Register File, completing the cycle.

### Custom Python Assembler

To support the custom architecture, I developed a dedicated Python assembler. This script parses human-readable DLX assembly files, resolves labels, and translates the instructions into machine code perfectly formatted for initialization within the FPGA's block RAM using Memory Initialization Files (.mif) for Quartus integration. An example program written in DLX is shown below. Because of the harvard architecture, two .mif files are created for the instruction memory and data memory, respectively.


```  title="Fib.dlx" linenums='1'
;Calculate the n-th Fibonacci number and store in memory	

.data	
    n		1	10 
    result	1	0

.text
	
	ADDI	R1, R0, 0
	ADDI	R2, R0, 1
	LW		R10, n(R0)
	
top
	SLEI	R11, R10, 1
	BNEZ	R11, exit
	ADD		R3, R1, R2
	ADDI	R1, R2, 0
	ADDI	R2, R3, 0
	SUBI	R10, R10, 1
	J		top
	
exit
	SW		result(R0), R3
	
done
	J		done
```


---

## Core Subsystems & Implementation

### Hazard Detection & Operand Forwarding
In a standard pipelined architecture, Read-After-Write (RAW) data dependencies introduce significant execution bottlenecks. Because the processor evaluates multiple instructions simultaneously, an instruction might attempt to read a register in the Decode (ID) stage before a previous instruction has reached the Writeback (WB) stage to update it. Without hardware mitigation, the pipeline would be forced to continuously flush or rely on the compiler to pad the execution with inefficient NOP (no-operation) instructions.

To maximize instruction throughput, I engineered a dedicated Hardware Forwarding Unit. This subsystem continuously unpacks the 32-bit instruction vectors across the pipeline stages and monitors the register addresses. If it detects that a destination register in the Execute/Memory (EX/MEM) or Memory/Writeback (MEM/WB) stages matches the source register of the instruction currently entering the ALU, it dynamically manipulates multiplexers to bypass the Register File. The required data is then routed directly from the downstream stages into the ALU input.

The logic natively prioritizes the most recent data (the MEM stage over the WB stage) and explicitly protects the hardwired zero register from being forwarded. The only exception is a Load-Use hazard. If an instruction requires data from an immediately preceding LW (Load Word) instruction, the Forwarding Unit coordinates with the Hazard Detection Unit to insert a strict 1-cycle stall, as the data is not physically available until the MEM stage completes.

```VHDL
-- Forwarding Unit: Instruction Unpacking & R-Type Logic Highlight
-- Extracted from forwarding_control.vhd

-- 1. Unpack instruction vectors into opcodes and register addresses
ex_op <= ex_instr_in(31 downto 26);
mem_op <= mem_instr_in(31 downto 26);
wb_op <= wb_instr_in(31 downto 26);
ex_reg1 <= ex_instr_in(20 downto 16);
ex_reg2 <= ex_instr_in(15 downto 11);
ex_rd <= ex_instr_in(25 downto 21);
mem_rd <= mem_instr_in(25 downto 21);
wb_rd <= wb_instr_in(25 downto 21);

control_logic : process(ex_op, mem_op, wb_op, ex_reg1, ex_reg2, mem_rd, wb_rd) begin
    -- Default state: Do not forward
    mux1_sel <= 3x"0"; 

    case ex_op is
        -- R-type Operations
        when OP_ADD | OP_SUB | OP_AND | OP_OR | OP_SLL | OP_SLT =>
            
            -- ALU Input A (Register 1) Forwarding
            if(ex_reg1 = mem_rd and ex_reg1 /= 5x"0") then
                -- Priority: Forward directly from EX/MEM stage
                mux1_sel <= 3x"1";
                
            elsif(ex_reg1 = wb_rd and ex_reg1 /= 5x"0") then
                -- Secondary: Forward from MEM/WB stage
                if wb_op = OP_LW then
                    mux1_sel <= 3x"3"; -- Specific routing for Load Word data
                else 
                    mux1_sel <= 3x"2"; 
                end if;
            else
                -- Read standard operand from ID/EX Register File
                mux1_sel <= 3x"0";
            end if;
            
            -- (Logic repeats for ALU Input B / ex_reg2) and other instruction types
```

The new execute stage with the created forwarding paths and the corresponding muxes controlled by the above logic can be seen below. 

![New Exectute Stage with Forwarding](/assets/images/executestage_new.png)
/// caption
Updated Execute stage with Forwarding Paths
///

### Hardware-Based Branch Prediction
The initial architecture of the processor relied on a naïve "assume not taken" approach for control flow, which severely bottlenecked performance on complex loops by introducing frequent pipeline flushes. To maximize instruction throughput and introduce speculative execution, I engineered a hardware branch predictor directly into the Instruction Fetch (IF) stage.

The prediction logic utilizes a Branch Target Buffer (BTB) and a Branch History Table (BHT) driven by a 2-bit saturating counter. By indexing these tables with the current Program Counter, the processor evaluates historical branch behavior (ranging from 00 strongly not taken to 11 strongly taken) and dynamically predicts control flow. This speculative target is multiplexed into the PC, allowing the pipeline to fetch subsequent instructions without waiting for the Execute (EX) stage to fully resolve the condition.

![New Fetch stage w/ predictive branching](/assets/images/new_fetchstage.png)
/// caption
Updated Fetch Stage with Predictive Branching
///


#### Critical Path Optimization & Recovery
The most significant challenge in implementing this speculative execution was managing the critical path for misprediction recovery. Initially, validating a branch prediction required a long, nested dependency chain waiting on the EX stage's Zero module to evaluate the arithmetic condition.

To achieve a higher overall pipeline clock frequency, I decoupled this logic. By verifying the correct address immediately based on the alu_b input, I reduced the misprediction check to a single hardware XOR gate (predicted_taken XOR concrete_not_taken). If a misprediction is flagged, the pipeline seamlessly corrects the PC to the concrete recovery address, flushes the invalid instructions from the Decode and Execute stages, and passes signals (branch_addr, is_branch, taken_branch) back to the Fetch stage to update the BHT.

#### Performance Outcomes

- **Execution Speedup:** Integrating speculative execution reduced the runtime of a computationally expensive Least Common Multiple (LCM) benchmark from 4 minutes and 3 seconds down to 2 minutes and 51 seconds—achieving a 41.8% performance increase at an identical clock speed.

- **Timing Closure:** By optimizing the misprediction critical path with the XOR decoupling, I eliminated setup and hold violations. This allowed the pipeline's overall Fmax to be successfully pushed to 70MHz on the DE10-Lite FPGA via a custom dual-clock Phase Locked Loop (PLL).


### UART Integration & Clock Domain Crossing
To support interactive terminal I/O, the processor integrates custom UART receiver and transmitter peripherals. However, increasing the pipeline's overall clock frequency to maximize computational performance introduced a critical hardware challenge: the initial UART modules were designed around a strict baud rate derived from the board's baseline 50MHz clock. Pushing the pipeline to 70MHz immediately broke this synchronous communication.

To resolve this without sacrificing processor speed, I architected a dual-clock system. I utilized a Phase Locked Loop (PLL) on the FPGA to generate a stable 50MHz clock dedicated to the UART peripherals, running completely parallel to the 70MHz pipeline clock. To bridge these asynchronous domains and prevent metastability, I engineered cross-clock domain (CDC) dual-clock FIFOs for both the input and output data paths.

#### Print Module Architecture & I/O Optimization

Managing the data transfer to the UART transmitter required a dedicated Print Module capable of interpreting different instruction types and managing pipeline stalls. 

![Print Module Architecture](/assets/images/printmodule.drawio.png)
/// caption
Data flow of the custom Print Module, illustrating ASCII conversion and buffer-driven pipeline stalling.
///

The module evaluates incoming opcodes and payloads packaged into the `CMD` signal and stored in a buffer to distinguish between printing raw ASCII characters (`PCH`) or numeric decimal values (`PD`). If a decimal print is requested, the data routes through a custom Number-to-String Converter to generate the correct ASCII byte sequence. A multiplexer then selects the appropriate ASCII character and write-enable signals, feeding them into the Print Buffer. The buffer subsequently passes the data to the UART Transmitter (`UART TX`) to drive the physical serial line.

If the processor attempts to write data faster than the UART can transmit it, filling the command buffer, the buffer asserts a `cmd_full` signal. This safely halts the primary 70MHz pipeline to prevent data loss. 

During the final empirical benchmarking (calculating the Least Common Multiple of large integers), this exact mechanism revealed a significant I/O bottleneck. The initial Print Buffer depth was limited to 256 bytes, which led to consistent stall assertions during print-heavy execution. By significantly expanding the depth of this output buffer, I eliminated the hardware stalls, achieving a near-zero execution time penalty for I/O operations.


#### Input Module Architecture & Data Conversion

Conversely, receiving dynamic user input from the terminal required an Input Module capable of assembling sequential, asynchronous keystrokes into cohesive 32-bit data words for the `GD` (Get Decimal Signed)  or `GDU` (Get Decimal Unsigned) instruction.

![Input Module Architecture](/assets/images/input.drawio.png)
/// caption
Data flow of the Input Module
///

When a `GD` instruction is decoded, the Input Module immediately asserts an `input_stall` signal, halting the primary processor pipeline. It then begins polling the UART Receiver's FIFO (Input Buffer) for incoming data. 

Because a user types in human-readable characters, the module must operate as a hardware-level string-to-integer converter. As each byte is read from the buffer, the state machine evaluates the character. If it is a numeric digit, the module converts the ASCII byte to its decimal equivalent, multiplies the current running total by 10, and adds the new value. 

This accumulator loop continues to process incoming characters until the user presses the `Enter` key (Carriage Return). Once the termination character is detected, the assembled 32-bit signed (or unsigned) integer is forwarded to the processor's register file, and the pipeline stall is safely released. This design successfully decouples the slow, unpredictable timing of human typing from the strict 70MHz execution cycle of the main pipeline.


---

## Verification & Hardware Analysis

Achieving timing closure and successful compilation is only half the battle in FPGA design. To guarantee the processor's reliability, the architecture underwent a rigorous two-phase verification process: simulated testbench validation followed by physical hardware analysis.

### Questa Advanced Simulation

Throughout the development cycle, every individual module—from the Hazard Detection Unit to the custom Print Buffer—was validated using ModelSim/Questa Advanced Simulator. 

I developed a comprehensive suite of custom DLX assembly programs designed specifically to trigger architectural edge cases. These testbenches intentionally forced consecutive Read-After-Write (RAW) hazards, Load-Use dependencies, and deep loop mispredictions. By analyzing the resulting waveforms, I was able to verify that the forwarding multiplexers bypassed the register file correctly and that the pipeline flushed cleanly without corrupting the state of the data memory. 

### Physical Hardware Validation (DE10-Lite)

Simulation parity with physical hardware was guaranteed by synthesizing the final design onto the DE10-Lite FPGA. To validate the processor's internal state during live execution, I utilized Intel's Signal Tap Logic Analyzer, probing internal pipeline registers to monitor the real-time behavior of the UART Clock Domain Crossing FIFOs.

![Hardware Execution & Benchmarking](/assets/images/fpga-execution-putty.png)
/// caption
Physical hardware validation: The 7-segment displays acting as a hardware timer (bottom) during the execution of a computationally heavy Least Common Multiple program, with output sent via UART to a PuTTY terminal (top)
///

To accurately benchmark the 70MHz processor against complex workloads, I engineered a custom hardware timer peripheral. Driven by three new custom assembly instructions (`TR` Timer Reset, `TGO` Timer Go, `TSP` Timer Stop), the timer leverages the DE10-Lite's 7-segment displays to track execution time down to the millisecond. 

Running a heavy Least Common Multiple (LCM) algorithm on the physical board successfully computed the result for massive integer inputs, transmitting the correct values back to a PC via the UART terminal. This physical benchmarking proved the complete stability of the 5-stage pipeline, the branch predictor, and the asynchronous I/O peripherals under continuous, heavy execution.

---

## Source Code & Final Outcomes

The final architecture represents a complete, highly optimized softcore processor capable of compiling and executing the full DLX instruction set. 

* **Target Hardware:** DE10-Lite FPGA (MAX 10)
* **Maximum Clock Frequency:** 70 MHz (via custom PLL)

[View GitHub Repository](https://github.com/masondavis9){ .md-button .md-button--primary }