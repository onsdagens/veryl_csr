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

A peripheral typically comprise several (32-bit) registers, where each register can be broken down into several fields. This is supported by our CSR abstraction by allowing a single backing store (register) to be accessed from different views (CSR addresses), each representing a field of said register. This allows per-field single cycle atomic access, while ensuring field isolation (i.e., operations are ensured by the hardware abstraction to *only* affect the desired field, while neighboring fields are unaffected). Besides improved safety, our CSR abstraction provides a set of performance enhancements.

The immediate CSR operations are limited to 5-bit fields (by the [rv32-isa]), while wider accesses are done through register `rs1`. For the latter, a CSR view can be configured to pre-shift the register content (thus eliminating the software overhead of shifting).

Similarly, for reads (to `rd`), a CSR view can be configured to post-shift the field (thus also in this case eliminate the software overhead of shifting).

Our design allows for agile hardware/software co-design, where the peripheral and driver are developed in tandem. Only at this point the access patterns will come clear, and the appropriate peripheral can be be devised. 


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
    param Addr      : CsrAddr = 0 , // 12 bit address for the CSR
    param ImmOffset : u32     = 0 , // offset for immediate write field
    param ImmWidth  : u32     = 5 , // immediate write field width
    param ReadOffset: u32     = 0 , // read field offset
    param ReadWidth : u32     = 32, // read field width
    param WriteShift : logic   = 1 , // when 1, i_rs1_data is shifted to field offset 
    param ReadShift : logic   = 1 , // when 1, read field shifted to offset 0
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

The CSR is configured by its address (`CsrAddr`), field write offset (0..=31), and field width (1..=5), where the latter two are dedicated for the immediate operations. Additionally, field read offset (0..=31) and field width (0..=32) can be declared. These allow for single cycle CSR field read operations, where the result is optionally shifted to offset 0. 

As inputs the CSR takes the enable (`i_csr_ena` assumed true if instruction is decade as a CSR operation), the 12 bit address (`CsrAddr`), the 32 input data for register based write operations, and the 5 bit input for immediate operations, along with the opcode (`i_csr_op`).

Interaction with the backing storage is done through the `reg` input (`i_reg_data`), write enable (`o_reg_w_ena`) and corresponding data to write (`o_reg_data`). Similarly reads are handled by read enable (`o_reg_r_ena`) and corresponding data (`o_reg_r_data`). 

The implementation ensures field isolation, i.e., immediate operations will *only* affects the range as defined by offset and width.

### Functional specification

The CSR operations follow section 6.1 of [rv32-isa].

### Implementation details

The `o_reg_r_ena` signal indicates that a read operation was performed, allowing the wrapping instance (e.g., a UART peripheral) to introduce side effects (e.g., clearing data ready bit, on read). The `o_reg_r_data` takes the value 0 in case a read to the specific CSR was not performed. This allows effective bus-less collection of CSR output data by means of simple logic or between all CSR `o_reg_r_data` (no requirement to multiplexer trees). For ASIC implementations, a CSR data bus design can leverage the `o_reg_r_ena` signal to control a tri-state driver connecting all CSRs.

---

## Example

![CSR][csr_fields]

The `src/csr_fields.veryl` provides an example of implementing a peripheral at address `Addr` 0x101 (a 5 bit field, at offset 0), `Addr + 1` (a 4 bit field at offset 8), and `Addr + 2` (a 2 bit field at offset 12).

A single 32 bit backing storage is shared among the field views, allowing a mix of register and field based operations.

## Extendability

As seen en the example above, the backing data is *owned* by the wrapping component. This allows the wrapper to provide additional interfaces, e.g., allowing concurrent updates to the underlying register(s). This is useful to implement peripherals, e.g., a flag bit signal can be raised by the hardware and consecutively cleared by the software as a side effect or reading or explicit clear.

Notice, the wrapper component is responsible for resolving race conditions between hardware and software accesses.

## Limitations

The current implementation does not implement read only operations.

## Test

```shell
veryl test --wave --verbose
surfer src/csr_fields.vcd
```

[csr]: /images/csr.svg
[csr_fields]: /images/csr_fields.svg
[rv32-isa]: https://drive.google.com/file/d/1uviu1nH-tScFfgrovvFCrj7Omv8tFtkp/view?usp=drive_link
