

# RandomX
RandomX ("random ex") is an experimental proof of work (PoW) algorithm that uses random code execution to achieve ASIC resistance.

RandomX uses a simple low-level language (instruction set), which was designed so that any random bitstring forms a valid program.

*Software implementation details and design notes are written in italics.*

## Virtual machine
RandomX is intended to be run efficiently and easily on a general-purpose CPU. The virtual machine (VM) which runs RandomX code attempts to simulate a generic CPU using the following set of components:

![Imgur](https://i.imgur.com/dRU8jiu.png)

#### DRAM
The VM has access to 4 GiB of external memory in read-only mode. The DRAM memory blob is generated from the hash of the previous block using AES encryption (TBD). The contents of the DRAM blob change on average every 2 minutes. The DRAM blob is read with a maximum rate of 2.5 GiB/s per thread.

*The DRAM blob can be generated in 0.1-0.3 seconds using 8 threads with hardware-accelerated AES and dual channel DDR3 or DDR4 memory. Dual channel DDR4 memory has enough bandwidth to support up to 16 mining threads.*

#### MMU
The memory management unit (MMU) interfaces the CPU with the DRAM blob. The purpose of the MMU is to translate the random memory accesses generated by the random program into a DRAM-friendly access pattern, where memory reads are not bound by access latency. The MMU accepts a 32-bit address `addr` and outputs a 64-bit value from DRAM. The MMU splits the 4 GiB DRAM blob into 256-byte blocks. Data within one block is always read sequentially in 32 reads (32×8 bytes). When the current block has been consumed, reading jumps to a random block. The address of the next block is calculated 16 reads before the current block is exhausted to enable efficient prefetching. The MMU uses three internal registers:
* **m0** - Address of the next quadword to be read from memory (32-bit, 8-byte aligned).
* **m1** - Address of the next block to be read from memory (32-bit, 256-byte aligned).
* **mx** - Random 32-bit counter that determines the address of the next block. After each read, the read address is mixed with the counter: `mx ^= addr`. When the 16th quadword of the current block is read (the value of the `m0` register ends with `0x80`), the value of the `mx` register is copied into register `m1` and the last 8 bits of `m1` are cleared.

*When the value of the `m1` register is changed, the memory location can be preloaded into CPU cache using the x86 `PREFETCH` instruction or ARM `PRFM` instruction. Implicit prefetch should ensure that sequentially accessed memory is already in the cache.*

#### Scratchpad
The VM contains a 256 KiB scratchpad, which is accessed randomly both for reading and writing. The scratchpad is split into two segments (16 KiB and 240 KiB). 75% of accesses are into the first 16 KiB.

*The scratchpad access pattern mimics the usual CPU cache structure. The first 16 KiB should be covered by the L1 cache, while the remaining accesses should hit the L2 cache. In some cases, the read address can be calculated in advance (see below), which should limit the impact of L1 cache misses.*

#### Program
The actual program is stored in a 8 KiB ring buffer structure. Each program consists of 512 random 128-bit instructions. The ring buffer structure makes sure that the program forms a closed infinite loop.

*For high-performance mining, the program should be translated directly into machine code. The whole program should fit into the L1 instruction cache and hot execution paths should stay in the µOP cache that is used by newer x86 CPUs. This should limit the number of front-end stalls and keep the CPU busy most of the time.*

#### Control unit
The control unit (CU) controls the execution of the program. It reads instructions from the program buffer and sends commands to the other units. The CU contains 3 internal registers:
* **pc** - Address of the next instruction in the program buffer to be executed (64-bit, 8 byte aligned).
* **sp** - Address of the top of the stack (64-bit, 8 byte aligned).
* **ic** - Instruction counter contains the number of instructions to execute before terminating. The register is decremented after each instruction and the program execution stops when `ic` reaches `0`.

*Fixed number of executed instructions per program should ensure roughly equal runtime of each random program.*

#### Stack
To simulate function calls, the VM uses a stack structure. The program interacts with the stack using the CALL and RET instructions. The stack has unlimited size and each stack element is 64 bits wide.

#### Register file
The VM has 8 integer registers `r0`-`r7` and 8 floating point registers `f0`-`f7`. All registers are 64 bits wide.

*The number of registers is low enough so that they can be stored in actual hardware registers on most CPUs.*

#### ALU
The arithmetic logic unit (ALU) performs integer operations. The ALU can perform binary integer operations from 11 groups (ADD, SUB, MUL, DIV, AND, OR, XOR, SHL, SHR, ROL, ROR) with operand sizes of 64 or 32 bits.

#### FPU
The floating-point unit performs IEEE-754 compliant math using 64-bit double precision floating point numbers.

#### Endianness
The VM stores and loads all data in little-endian byte order.

## Instruction set
The 128-bit instruction is encoded as follows:

![Imgur](https://i.imgur.com/thpvVHN.png)

#### Opcode
There are 256 opcodes, which are distributed between various operations depending on their weight (how often they will occur in the program on average). The distribution of opcodes is following (TBD):

|operation|number of opcodes||
|---------|-----------------|----|
|ALU operations|158|61.7%|
|FPU operations|66|25.8%|
|Control flow |32|12.5%|

#### Operand A
The first operand is read from memory. The location is determined by the `loc(a)` flag:

|loc(a)[2:0]|read A from|address size (W)
|---------|-|-|
|000|DRAM|32 bits|
|001|DRAM|32 bits|
|010|DRAM|32 bits|
|011|DRAM|32 bits|
|100|scratchpad|15 bits|
|101|scratchpad|11 bits|
|110|scratchpad|11 bits|
|111|scratchpad|11 bits|

Flag `reg(a)` encodes an integer register `r0`-`r7`.  The read address is calculated as:
```
reg(a) ^= addr0
addr(a) = reg(a)[W-1:0]
```
For reading from the scratchpad, `addr(a)` is multiplied by 8 for 8-byte aligned access.

#### Operand B
The second operand is loaded either from a register or from an immediate value encoded within the instruction. The `reg(b)` flag encodes an integer register (ALU operations) or a floating point register (FPU operations).

|loc(b)[2:0]|read B from|
|---------|-|
|000|register `reg(b)`|
|001|register `reg(b)`|
|010|register `reg(b)`|
|011|register `reg(b)`|
|100|register `reg(b)`|
|101|register `reg(b)`|
|110|`imm0` or `imm1`|
|111|`imm0` or `imm1`|

`imm0` is an 8-bit immediate value, which is used for shift and rotate ALU operations.

`imm1` is a 32-bit immediate value which is used for most operations. For operands larger than 32 bits, the value is zero-extended for unsigned instructions and sign-extended for signed instructions. For FPU instructions, the value is treated as a signed 32-bit integer and converted to a double precision floating point format.

#### Operand C
The third operand is the location where the result is stored.

|loc\(c\)[2:0]|write C to|address size (W)
|---------|-|-|
|000|scratchpad|15 bits|
|001|scratchpad|11 bits|
|010|scratchpad|11 bits|
|011|scratchpad|11 bits|
|100|register `reg(c)`|-|
|101|register `reg(c)`|-|
|110|register `reg(c)`|-|
|111|register `reg(c)`|-|

The `reg(c)` flag encodes an integer register (ALU operations) or a floating point register (FPU operations).  For writing to the scratchpad, an integer register is always used and the write address is calculated as:
```
addr(c) = (addr1 ^ reg(c))[W-1:0] * 8
```
*CPUs are typically designed for a 2:1 load:store ratio, so each VM instruction performs on average 1 memory read and 0.5 write to memory.*

#### imm0
An 8-bit immediate value that is used as the shift/rotate count by some ALU instructions and as the jump offset of the CALL instruction.

#### addr0
A 32-bit address mask that is used to calculate the read address for the A operand.

#### addr1
A 32-bit address mask that is used to calculate the write address for the C operand. `addr1` is equal to `imm1`.

### ALU instructions

|opcodes|instruction|signed|A width|B width|C|C width|
|-|-|-|-|-|-|-|
|0-13|ADD_64|no|64|64|A + B|64|
|14-20|ADD_32|no|32|32|A + B|32|
|21-34|SUB_64|no|64|64|A - B|64|
|35-41|SUB_32|no|32|32|A - B|32|
|42-45|MUL_64|no|64|64|A * B|64|
|46-49|MULH_64|no|64|64|A * B|64|
|50-53|MUL_32|no|32|32|A * B|64|
|54-57|IMUL_32|yes|32|32|A * B|64|
|58-61|IMULH_64|yes|64|64|A * B|64|
|62|DIV_64|no|64|32|A / B|32|
|63|IDIV_64|yes|64|32|A / B|32|
|64-76|AND_64|no|64|64|A & B|64|
|77-82|AND_32|no|32|32|A & B|32|
|83-95|OR_64|no|64|64|A &#124; B|64|
|96-101|OR_32|no|32|32|A &#124; B|32|
|102-115|XOR_64|no|64|64|A ^ B|64|
|116-121|XOR_32|no|32|32|A ^ B|32|
|122-128|SHL_64|no|64|6|A << B|64|
|129-132|SHR_64|no|64|6|A >> B|64|
|133-135|SAR_64|yes|64|6|A >> B|64|
|136-146|ROL_64|no|64|6|A <<< B|64|
|147-157|ROR_64|no|64|6|A >>> B|64|

##### 32-bit operations
Instructions ADD_32, SUB_32, AND_32, OR_32, XOR_32 only use the low-order 32 bits of the input operands. The result of these operations is 32 bits long and bits 32-63 of C are zero.

##### Multiplication
There are 5 different multiplication operations. MUL_64 and MULH_64 both take 64-bit unsigned operands, but MUL_64 produces the low 64 bits of the result and MULH_64 produces the high 64 bits. MUL_32 and IMUL_32 use only the low-order 32 bits of the operands and produce a 64-bit result. The signed variant interprets the arguments as signed integers. IMULH_64 takes two 64-bit signed operands and produces the high-order 64 bits of the result.

The following table shows an example of the output of the 5 multiplication instructions for inputs `A = 0xBC550E96BA88A72B` and `B = 0xF5391FA9F18D6273`:

|instruction|A interpreted as|B interpreted as|result C|
|-|-|-|-|
|MUL_64|13570769092688258859|17670189427727360627|`0x28723424A9108E51`|
|MULH_64|13570769092688258859|17670189427727360627|`0xB4676D31D2B34883`|
|MUL_32|3129517867|4052574835|`0xB001AA5FA9108E51`|
|IMUL_32|-1165449429|-242392461|`0x03EBA0C1A9108E51`|
|IMULH_64|-4875974981021292757|-776554645982190989|`0x02D93EF1269D3EE5`|

##### Division
For the division instructions, the dividend is 64 bits long and the divisor 32 bits long. The IDIV_64 instruction interprets both operands as signed integers. In case of division by zero or signed overflow, the result is equal to the dividend `A`.

*Division by zero can be handled without branching by conditional move (`IF B == 0 THEN B = 1`). Signed overflow happens only for the signed variant when the minimum negative value is divided by -1. In this extremely rare case, ARM produces the "correct" result, but x86 throws a hardware exception, which must be handled.*

##### Shift and rotate
The shift/rotate instructions use just the bottom 6 bits of the `B` operand (`imm0` is used as the immediate value). All treat `A` as unsigned except SAR_64, which performs an arithmetic right shift by copying the sign bit.

### FPU instructions

|opcodes|instruction|C|
|-|-|-|
|158-175|FADD|A + B|
|176-193|FSUB|A - B|
|194-211|FMUL|A * B|
|212-214|FDIV|A / B|
|215-221|FSQRT|sqrt(A)|
|222-223|FROUND|A|

FPU instructions conform to the IEEE-754 specification, so they must give correctly rounded results. Initial rounding mode is RN (Round to Nearest). Denormal values may not be produced by any operation.

*Denormals can be disabled by setting the FTZ flag in x86 SSE and ARM Neon engines. This is done for performance reasons.*

Operands loaded from memory are treated as signed 64-bit integers and converted to double precision floating point format. Operands loaded from floating point registers are used directly.

##### FSQRT
The sign bit of the FSQRT operand is always cleared first, so only non-negative values are used.

*In x86, the `SQRTSD` instruction must be used. The legacy `FSQRT` instruction doesn't produce correctly rounded results in all cases.*

##### FROUND
The FROUND instruction changes the rounding mode for all subsequent FPU operations depending on the two right-most bits of A:

|A[1:0]|rounding mode|
|-------|------------|
|00|Round to Nearest (RN) mode|
|01|Round towards Plus Infinity (RP) mode
|10|Round towards Minus Infinity (RM) mode
|11|Round towards Zero (RZ) mode

*The two-bit flag value exactly corresponds to bits 13-14 of the x86 `MXCSR` register and bits 22-23 of the ARM `FPSCR` register.*

### Control flow instructions
The following 2 control flow instructions are supported:

|opcodes|instruction|function|
|-|-|-|
|224-240|CALL|near procedure call|
|241-255|RET|return from procedure|

Both instructions are conditional in 75% of cases. The jump is taken only if `B <= imm1`. For the 25% of cases when `B` is equal to `imm1`, the jump is unconditional. In case the branch is not taken, both instructions become "arithmetic no-op" `C = A`.

##### CALL
Taken CALL instruction pushes the values `A` and `pc` (program counter) onto the stack and then performs a forward jump relative to the value of `pc`. The forward offset is equal to `16 * (imm0[7:0] + 1)`. Maximum jump distance is therefore 128 instructions forward (this means that at least 4 correctly spaced CALL instructions are needed to form a loop in the program).

##### RET
The RET instruction behaves like "not taken" when the stack is empty. Taken RET instruction pops the return address `raddr` from the stack (it's the instruction following the previous CALL), then pops a return value `retval` from the stack and sets `C = A ^ retval`. Finally, the instruction jumps back to `raddr`.

## Program generation
The program is initialized from a 256-bit seed value `S`.
1. A [pcg32](http://www.pcg-random.org/)  random number generator is initialized with state `S[63:0]`.
2. The generator is used to generate random 128 bytes `R1`.
3. Integer registers `r0`-`r7` are initialized using bytes 0-63 of `R1`.
4. Floating point registers `f0`-`f7` are initialized using bytes 64-127 of `R1` interpreted as 8 64-bit signed integers converted to a double precision floating point format.
5. The initial value of the `m0` register is set to `S[95:64]` and the the last 8 bits are cleared (256-byte aligned).
6. `S` is expanded into 10 AES round keys `K0`-`K9`.
7. `R1` is exploded into a 264 KiB buffer `B` by repeated 10-round AES encryption.
8. The scratchpad is set to the first 256 KiB of `B`.
9. The program buffer is set to the final 8 KiB of `B`.
10. The remaining registers are initialized as `pc = 0`, `sp = 0`, `ic = 1048576` (TBD), `mx = 0`.


## Result
When the program terminates (the value of `ic` register reaches 0), the final result is calculated as follows:
1. The register file is treated as a 128-byte value `R2`.
3. The 256 KiB scratchpad is imploded into a 128-byte digest `D` using 10-round AES decryption with keys `K0`-`K9` and XORing each 128-byte chunk with `R2`.
4. `D` is hashed using the Blake2b 256-bit hash function. This is the result of the PoW.

*The stack is not included in the result calculation to enable platform-specific return addresses.*

### Chaining
The program generation, execution and result calculation can be chained multiple times to discourage mining strategies that search for programs with particular properties.
