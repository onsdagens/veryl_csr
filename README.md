# CSR - RISC-V Control Status Register

Highly configure CSR abstraction, featuring:

- Atomic read/write register and field accesses leveraging the RISC-V [rv32-isa] specification.
- Efficient single cycle read/write field overlays with:
  - enforced field isolation.
  - optional pre/post shifting, reducing software overhead.
- Self address selection for effective FPGA bus-less CSR implementations.

---

## General architecture

The RISC-V RV32 ISA defines a 12 bit address space for CSR registers, along with an enumerated set of read/write atomic operations. These CSRs provide a means to efficiently interact with the underlying hardware. The RISC-V specification, defines a set of core peripherals with specified CSR mappings.

In context of embedded real-time systems, CSR based peripherals offer an interesting alternative to traditional MMIO implementations. From a software perspective, driver implementations can leverage per register-field single cycle atomic access, while from a hardware perspective our CSR abstraction offers precise control over side effects and race conditions.

A peripheral typically comprise several (32-bit) registers, where each register can be broken down into several fields. This is supported by our CSR abstraction by allowing a single backing store (register) to be accessed from different views (CSR addresses), each representing a field of said register. This allows per-field single cycle read/write atomic access with hardware enforced field isolation (i.e., operations are ensured to *only* affect the desired field, while neighboring fields are unaffected). Besides improved safety, our CSR abstraction provides a set of performance enhancements.

The immediate CSR operations are limited to 5-bit fields (by interpreting the `rs1` register number as a 5-bit *constant*), while wider accesses are done based on the *value* of register `rs1`. For the latter, a CSR view can be configured to pre-shift the register content (thus eliminating the software overhead of shifting and masking).

Similarly, for reads (to `rd`), a CSR view can be configured to post-shift the field (thus also in this case eliminate the software overhead of shifting and masking).

Our design allows for agile hardware/software co-design, where the peripheral hardware and driver software are developed in tandem. Only at this point the access patterns will come clear, and the appropriate peripheral can be be devised.

![CSR][csr]

### Implementation

The Veryl CSR operation encoding is found in the `src/csr_pkg.veryl`:

```sv
package CsrPkg {

    enum CsrOp: logic<3> {
        ECALL = 3'b000, // EBREAK = 3'b000,
        CSRRW = 3'b001,
        CSRRS = 3'b010,
        CSRRC = 3'b011,
        CSRRWI = 3'b101,
        CSRRSI = 3'b110,
        CSRRCI = 3'b111,
    }

    type CsrAddr = logic<12>;
    type Reg     = logic<5> ;
}
```

For the CSR implementation, `src/csr.veryl` provides a general abstraction:

```sv
module Csr #(
    param Addr       : CsrAddr = 0          , // 12 bit address for the CSR (view)
    param FieldOffset: u32     = 0          , // offset for field
    param FieldWidth : u32     = 32         , // field width
    param ImmOffset  : u32     = FieldOffset, // immediate access
    param ImmWidth   : u32     = FieldWidth ,
    param RegOffset  : u32     = FieldOffset, // register access
    param RegWidth   : u32     = FieldWidth ,

    param WriteShift: logic = 0, // when 1, i_rs1_data is left shifted by FieldOffset
    param ReadShift : logic = 1, // when 1, o_reg_r_data is right shifted by FieldOffset 
) (
    i_csr_ena : input logic      ,
    i_csr_addr: input CsrAddr    ,
    i_rs1_data: input logic  <32>,
    i_rs1     : input Reg        ,
    i_rd      : input Reg        ,
    i_csr_op  : input CsrOp      ,

    i_reg_data  : input  logic<32>,
    o_reg_w_ena : output logic    ,
    o_reg_w_data: output logic<32>,
    o_reg_r_ena : output logic    ,
    o_reg_r_data: output logic<32>,
)
```

The CSR is configured by its address (`CsrAddr`), field offset (`Offset`, 0..=31), and field width (`Width`, 1..=5). For immediate operations, the field access width can be uniquely defined, allowing e.g., exposing the complete 32 bits for register based read/write access, while disabling immediate field access.

As inputs the CSR takes the enable (`i_csr_ena` assumed true if instruction is decade as a CSR operation), the 12 bit address (`CsrAddr`), the 32 input data for register based write operations, and the 5 bit input for immediate operations, along with the opcode (`i_csr_op`).

Interaction with the backing storage is done through the register data input (`i_reg_data`), write enable (`o_reg_w_ena`) and corresponding data to write (`o_reg_data`). Similarly reads are handled by read enable (`o_reg_r_ena`) and corresponding data (`o_reg_r_data`).

The implementation ensures field isolation, i.e., immediate operations will *only* affect the range as defined by offset and width.

### Functional specification

The CSR operations follow section 6.1 of [rv32-isa].

### Implementation details

The `o_reg_r_ena` signal indicates that a read operation was performed, allowing the wrapping instance (e.g., a UART peripheral) to introduce side effects (e.g., clearing data ready bit, on read). The `o_reg_r_data` takes the value 0 in case a read to the specific CSR was not performed. This allows effective bus-less collection of CSR output data by means of simple logic or between all CSR `o_reg_r_data` (no requirement to multiplexer trees). For ASIC implementations, a CSR data bus design can leverage the `o_reg_r_ena` signal to control a tri-state driver connecting all CSRs.

---

## Example

![CSR][csr_fields]

The `src/csr_fields.veryl` provides an example of implementing a peripheral at address `Addr` 0x101 (a 5 bit field, at offset 0), `Addr + 1` (a 4 bit field at offset 8), and `Addr + 2` (a 2 bit field at offset 12).

A single backing storage is shared among the field views, allowing a mix of register and field based operations.

The peripheral implementation provides the backing store along with signals for field interaction:

```sv
    var reg_data: logic<32>; // backing store

    // signals for interaction with the base register
    var reg_w_ena : logic    ; // set 1 in case the field view were written to
    var reg_w_data: logic<32>; // the write value
    var reg_r_ena : logic    ; // set 1 in case the field view was read
    var reg_r_data: logic<32>; // the read value

    // interaction signals for each CSR view
    var reg_w_ena0 : logic    ;
    ... 
```

Backing storage can be of arbitrary size, while the CSR view instances require 32 bit data (formed from an extension or slice of the backing store). For simplicity of the example we choose a 32 bit backing store.

Now we can instantiate the CSR views. For the base register view we disable immediate field access by setting the `ImmWidth` to 0.

```sv
    inst csr: Csr #(
        Addr       ,
        ImmWidth: 0, // 0 bit field, no immediate field access
    ) (
        i_csr_ena               ,
        i_csr_addr              ,
        i_rs1_data              ,
        i_rs1                   ,
        i_rd                    ,
        i_csr_op                ,
        i_reg_data  : reg_data  , // interaction signals for base view
        o_reg_w_ena : reg_w_ena ,
        o_reg_w_data: reg_w_data,
        o_reg_r_ena : reg_r_ena ,
        o_reg_r_data: reg_r_data,
    );
```

Other CSR views are instantiated accordingly, showcasing the use of pre/post shifting for register based accesses:

```sv
    // default, 5 bit field at offset 0
    inst csr0: Csr #(
        Addr       : (Addr as u32 + 1) as CsrAddr,
        FieldOffset: 0                           , // not needed
        FieldWidth : 5                           ,
    ) (
        ...
        o_reg_w_ena : reg_w_ena0 , // interaction signals for field 1.
        ...
    );

    // 4 bit field at offset 8
    inst csr1: Csr #(
        Addr       : (Addr as u32 + 2) as CsrAddr,
        FieldOffset: 8                           , // 4 bit field at offset 8
        FieldWidth : 4                           , 
        ReadShift  : 0                           , // no pre/post shifting
        WriteShift : 0                           ,
    ) (
        ... // interaction signals for field 2
    );

    // 2 bit field at offset 12
    inst csr2: Csr #(
        Addr       : (Addr as u32 + 3) as CsrAddr,
        FieldOffset: 12                          , // 2 bit field at offset 12
        FieldWidth : 2                           , 
        ReadShift  : 1                           , // pre/post shifting
        WriteShift : 1                           ,
    ) (
        ... // interaction signals for field 3
    );
```

The business logic for the CSR view interaction is defined by the clocked update and combinatorial output:

```sv
    always_ff {
        if i_reset {
            reg_data = ResetValue;
        } else if reg_w_ena { // CSR 100
            reg_data = reg_w_data & 'hffff_3f1f;
        } else if reg_w_ena0 { // CSR 101
            reg_data = reg_w_data0;
        } else if reg_w_ena1 { // CSR 102
            reg_data = reg_w_data1 | 'h8000_0000;
        } else if reg_w_ena2 & ~reg_r_ena2 { // CSR 103, write not read
            reg_data = reg_w_data2;
        } else if reg_w_ena2 & reg_r_ena2 { // CSR 103, write and read
            reg_data = reg_w_data2 & 32'h7fff_ffff;
        } else if reg_r_ena2 { // CSR 103, read only
            reg_data = reg_data & 32'h7fff_ffff;
        }
    }

    assign o_data = (reg_r_data & 'hffff_3f1f) | reg_r_data0 | reg_r_data1 | reg_r_data2;
```

In the clocked update we can introduce arbitrary side effects. For the example, we set a flag (bit 31) in case a write operation is conducted to CSR 102. This flag is cleared by reading CSR 103.

## Extendability

The CSR abstraction strives to strike the balance between ease of use and flexibility. To this end, the CSR abstraction proper caters for the ISA defined operations, and allows for compile time configuration (through the set of carefully chosen instantiation parameters and interaction signal bindings).

Based on the configuration the CSR implementation ensures field isolation preventing undefined behavior in case of mis-use. Moreover, interaction signals allows the peripheral developer to precisely tailor the behavior for each register as well as the peripheral as a whole.

As seen en the example above, the backing data is *owned* by the wrapping component. This allows the wrapping peripheral component to implement concurrent updates to the underlying register(s) from both hardware and software. E.g., a flag bit signal can be raised by the peripheral hardware and consecutively cleared by the software (as a side effect of reading some field, or as an effect of an explicit clear operation). The interaction signals are designed to provide the necessary information to resolve race conditions between hardware and software accesses.

## Test

```shell
veryl test --wave --verbose
surfer src/csr_fields.vcd
```

[csr]: /images/csr.svg
[csr_fields]: /images/csr_fields.svg
[rv32-isa]: https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link
