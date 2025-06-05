# CSR - RISC-V Control Status Register

Highly configure CSR abstraction, featuring:

- Enumerated register fields.
- Immediate field overlays with shared storage, ensuring field isolation.
- Self address selection for FGPA bus-less I/O.

---

## General architcure

![CSR][csr]

The RISC-V RV32 ISA defines a 12 bit address space for CSR registers, along with an enumerated set of read/write atomic operations.

The Veryl encoding is found in the `src/csr_pkg.veryl`:

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
    type CsrImm  = logic<5> ;
}
```

For the implemantion, `src/csr.veryl` provides a general abstraction for CSRs:

```sv
module Csr #(
    param Addr     : CsrAddr = 0, // 12 bit address for the CSR
    param ImmOffset: u32     = 0, // offset for immediate field 
    param ImmWidth : u32     = 5, // field width
) (
    i_csr_ena : input logic      ,
    i_csr_addr: input CsrAddr    ,
    i_32      : input logic  <32>,
    i_5       : input CsrImm     ,
    i_csr_op  : input CsrOp      ,

    i_reg_data : input  logic<32>,
    o_reg_w_ena: output logic    ,
    o_reg_data : output logic<32>,
)
```

The CSR is configured by its address (`CsrAddr`), field offset (0..=31), and field width (1..=5), where the latter two are dedicated for the immediate operations.

As inputs the CSR takes the enable (`i_csr_ena` assumed true if instruction is decade as a CSR operation), the 12 bit address (`CsrAddr`), the 32 input data for register based write operations, and the 5 bit input for immediate operations, along with the opcode (`i_csr_op`).

Interaction with the backing storage is done through the `reg` input (`i_reg_data`), write enable (`o_reg_w_ena`) and corresponding data to write (`o_reg_data`).

The implementation ensures field isolation, ensuring that immediate operations *only* affects the range as defined by offset and width.

---

## Example

![CSR][csr_fields]

The `src/csr_fields.veryl` provides an example of implementing a periheral at address `Addr` 0x101 (a 5 bit field, at offset 0), `Addr + 1` (a 4 bit field at offset 8), and `Addr + 2` (a 2 bit field at offset 12).

A single 32 bit backing storage is shared among the field views, allowing a mix of register and field based operations.

## Limitaitions

The current implementation does not implement read only operations.

## Test

```shell
veryl test --wave --verbose
surfer src/csr_fields.vcd
```

[csr]: /images/csr.svg
[csr_fields]: /images/csr_fields.svg
