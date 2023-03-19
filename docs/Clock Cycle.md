## Pulse 1
##### Program Counter incremented
- D-RPC-MemAdder is high. It will already have been high on the previous pulse so we do not have to worry about race conditions. This signal will route the output of the Mem Adder to the Program Counter's DFF.
- D-WRT-RPC will activate, causing the Program Counter to write.

##### First byte of instruction stored
- The output of RPC is always sent to the input of the EEPROM. This means that on the previous pulse, the EEPROM will already have been outputting the next instruction.
- The output of the EEPROM (which is the first byte of the next instruction) is sent to the data input of the Instruction Buffer.
- D-WRT-CIRFirst activates, causing the first DFF in the Instruction Buffer to write the output of the EEPROM, thus storing the first byte of the instruction.
- Since the Instruction Buffer has already been receiving the first instruction on the previous clock cycle, the potential for a race condition is greatly reduced. However, depending on the comparative memory access times of the DFF and the SRAM, a race condition is theoretically possible, because we are incrementing the program counter on the same pulse.


## Pulse 2
##### Second byte of instruction stored
- On the previous pulse, we incremented the program counter, so the EEPROM now outputs the second byte of the next instruction. In fact, it has been outputting it for the majority of the previous pulse.
- The output of the EEPROM (which is the second byte of the next instruction) is sent to the data input of the Instruction Buffer.
- D-WRT-CIRFirst activates, causing the first DFF in the Instruction Buffer to write the output of the EEPROM, thus storing the first byte of the instruction.
- There is no potential for a race condition here, because the Program Counter is not incremented. The main reason for the program counter not being incremented is that a low signal is required on a DFF before it is written to again, and the Program Counter DFF was already written to on the previous pulse. This means the signal would be sustained high over two pulses rather than creating a rising edge, which is required to write to the DFF.


## Pulse 3
##### Program Counter Incremented
- D-RPC-MemAdder is high. It will already have been high on the previous pulse so we do not have to worry about race conditions. This signal will route the output of the Mem Adder to the Program Counter's DFF.
- D-WRT-RPC will activate, causing the Program Counter to write.

##### First register selected
- This step involves selecting the first register needed by the instruction (indicated by the most significant nibble of the second byte).
- The Control Unit outputs the first register that needs to be selected in the Select signal. This is computed in the Operand Integrator with the help of D-CLK-SelectFirst. This signal is sent to the select input of the Register File.
- If no first register needs to be fetched (ie, D-CLK-SelectFirst is inactive), the Select signal is set to 0. This will cause the Register File in turn to output 0. We will still write this 0 to the Register Buffer, however, so that the register from the previous instruction does not stay there and potentially cause unexpected behaviour.
- The Register File forwards the contents of the register indicated by the select input to its data output.
- We do not actually store the data output of the Register File because this would cause a race condition, where the Register File must output the correct data before the Register Buffer DFF finishes (or even starts) writing.


## Pulse 4
##### First register stored
- The output of the Register File is sent to the Register Buffer. This was already the case for the majority of the previous pulse, preventing a race condition.
- D-WRT-RegBufferFirst is activated, causing the first DFF in the Register Buffer to store the Register Buffer input.
- We could potentially select the second register here, but this may cause a race condition. If another pulse is needed in future for whatever reason, this is a potential compromise that can be made, but it is important to check that it does not cause a race condition.


## Pulse 5
##### Second register selected
- This step involves selecting the second register needed by the instruction (indicated by the least significant nibble of the second byte).
- The Control Unit outputs the second register that needs to be selected in the Select signal. This is computed in the Operand Integrator with the help of D-CLK-SelectSecond. This signal is sent to the select input of the Register File.
- If no second register needs to be fetched (ie, D-CLK-SelectSecond is inactive), the Select signal is set to 0. This will cause the Register File in turn to output 0. We will still write this 0 to the Register Buffer, however, so that the register from the previous instruction does not stay there and potentially cause unexpected behaviour.
- The Register File forwards the contents of the register indicated by the select input to its data output.
- We do not actually store the data output of the Register File because this would cause a race condition, where the Register File must output the correct data before the Register Buffer DFF finishes (or even starts) writing.


## Pulse 6
##### Second register stored
- The output of the Register File is sent to the Register Buffer. This was already the case for the majority of the previous pulse, preventing a race condition.
- D-WRT-RegBufferSecond is activated, causing the second DFF in the Register Buffer to store the Register Buffer input.


## Pulse 7
##### Output register selected
- This step involves selecting the register that the instruction will output to (the second register).
- The Control Unit outputs the register that needs to be selected in the Select signal. This is computed in the Operand Integrator with the help of D-CLK-SelectSecond (the output register is always the second register). This signal is sent to the select input of the Register File.
- If no output register needs to be selected (ie, D-CLK-SelectSecond is inactive), the Select signal is set to 0. We will still send a write signal, but since the Register File has selected 0, no register will actually be written to.
- We do not actually store the data output of the Register File because this would cause a race condition, where we must select the correct register before writing.

##### Program Counter written to
- This will only occur if a branch instruction is invoked
- In an immediate branch, the value we are branching to will already have been propagated to the Program Counter in the previous few pulses via the Addressor if S-ADR-Immediate is active
- In a direct branch, the value we are branching to is the combined value of the two registers that are stored in the Register Buffer, which have already been propagated to the Program Counter in the previous few pulses via the Addressor if S-ADR-Direct is active.
- D-WRT-RPC will activate on this pulse, causing the Program Counter to write the value that has been propagated.
- This cannot happen on Pulse 6 because the second register is still being stored at this point, so this would trigger a race condition in direct branches, where the second register must be propagated before the Program Counter is written to.
- This cannot happen on Pulse 8 because the clock signal would be high over both Pulse 8 and the subsequent Pulse 1. This would result on a write not occuring on Pulse 1, because there is no rising edge that triggers the write. Additionally, there would be a race condition, as the output of the MEM Adder would have to be propagated to the Program Counter before the Program Counter was written to.


## Pulse 8
##### Flag Register written to
- This will only occur if an arithmetic, logic, comparison, or test instruction is invoked (if S-ALU-FlagWrite is active).
- The ALU will have already performed any computations that the instruction requires after the second register was fetched.
- The flags that the ALU outputs have already been propagated to the Flag Register for the last few pulses.
- D-WRT-Flags is active, which will write the propagated outputs of the ALU to the Flag Register.

##### Output Register written to
- This will only occur if the instruction outputs to a register (if S-WriteVAL, S-WriteMDR, or S-WriteALU).
- Data from the Control Unit's value output, the MDR, and the ALU will all be propagated to the Register File.
- However, only one of these will be propagated to the registers inside the Register FIle. If S-WriteVAL is high, this will be the Control Unit's value output. If S-WriteMDR is high, this will be the output of the MDR. If S-WriteALU is high, this will be the ALU's output. One of these three signals will be active over the entire cycle, so the data will already have been propagated to the registers inside the Register File in the previous pulse, preventing a race condition.
- The output register was already propagated in the previous pulse, preventing a race condition.
- D-WRT-RegOutput is active, which causes the data propagated to the registers to be written to the selected register.

##### MDR written to
- This will only occur if a load or move-to-MDR instruction (LDD, LDI, or MTD) is invoked.
- For a load instruction, the S-MEM-Store flag is offand so the SRAM will be in read mode. The address will already have been propagated to the SRAM. Thus, if a load instruction is being invoked, the SRAM will already be outputting the data stored at the specified address in the previous pulse, preventing a race condition.
- For a move-to-MDR instruction, S-MDR-WriteALU will already be active. In a move-to-MDR instruction, the ALU will propagate (without modification) the value we are moving to the MDR. This means the ALU output will already be propagated to the MDR during the previous pulse, preventing a race condition.
- The D-WRT-MDR flag is active, causing the MDR to write the value that has been propagated to it.

##### SRAM Written to
- This will only occur if a store instruction (STD or STI) is invoked.
- The S-MEM-Store flag will have put the SRAM into write mode already. The address will also have been propagated to it during the previous pulses. The S-MEM-Store flag will have also propagated the MDR to the SRAM during the previous pulses. This means everything required to do a memory write has already been accomplished in previous pulses, preventing a race condition.
- The D-WRT-SRAM flag is active, causing the SRAM to write the propagated data to the propagated address.