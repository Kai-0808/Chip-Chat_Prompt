## 07_Bias register
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Bias Register:

- Input:
  - clk: 1 bit. Clock signal.
  - rst_n: 1 bit. Asynchronous active low reset signal.
  - raddr: 5 bits. The address of the data to be read.
  - waddr: 5 bits. The address of the data to be written.
  - wr_en: 1 bit. Write enable signal. 0: inactive, 1: write the single pixel.
  - data_in: 32 bits. The data to be written.

- Output:
  - data_out: 32 bits. The data read from the register file.

- Functionality:
The register file has batch (x32) of 32-bit pixel, i.e., `logic[31:0] reg_file [0:31]`. The register file is reset to 0 when the `rst_n` signal is low.
`raddr/waddr` infers the address of image. The register file updates its values to the address specified by `waddr` every cycle when performing the write operation, and always sends the values to the `data_out` output from the address specified by `raddr`.

When `wr_en` is 0, the register file is inactive.
When `wr_en` is 1, the register file writes the single pixel from `data_in` to the address specified by `raddr`.

- Restriction:
  - RTL code should be able to be synthesized.
  - Separate the combinational and sequential logic block. 
  - Give the entire SystemVerilog RTL code without any shortcut.

### ChatGPT:
Certainly, based on the specification provided, here's the RTL code for the Bias Register:

```systemverilog
module bias_register (
    input logic clk,
    input logic rst_n,
    input logic [4:0] raddr,
    input logic [4:0] waddr,
    input logic wr_en,
    input logic [31:0] data_in,
    output logic [31:0] data_out
);

    // Register file declaration
    logic [31:0] reg_file [0:31];

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Resetting the register file to all zeros
            for (int i = 0; i < 32; i = i + 1) begin
                reg_file[i] <= 32'b0;
            end
        end else if (wr_en) begin
            // Writing to the register file
            reg_file[waddr] <= data_in;
        end
    end

    // Combinational block for reading the data out
    always_comb begin
        data_out = reg_file[raddr];
    end

endmodule
```

Notes:
1. The module follows the provided specification.
2. The logic is separated into sequential and combinational blocks.
3. The register file is 32 x 32-bits, as specified.
4. An asynchronous active low reset is used to initialize the register file to zero. If reset is not active, and the write enable (`wr_en`) is set, the data is written into the register file at the specified write address.
5. The read address (`raddr`) is directly used in the combinational block to output the data.

### User:
Reduce the addr to 3-bit and change the memory depth to 8.

### ChatGPT:
Sure, here's the modified RTL code for the Bias Register based on the updated specifications:

```systemverilog
module bias_register (
    input logic clk,
    input logic rst_n,
    input logic [2:0] raddr,
    input logic [2:0] waddr,
    input logic wr_en,
    input logic [31:0] data_in,
    output logic [31:0] data_out
);

    // Register file declaration
    logic [31:0] reg_file [0:7]; // Memory depth changed to 8

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Resetting the register file to all zeros
            for (int i = 0; i < 8; i = i + 1) begin
                reg_file[i] <= 32'b0;
            end
        end else if (wr_en) begin
            // Writing to the register file
            reg_file[waddr] <= data_in;
        end
    end

    // Combinational block for reading the data out
    always_comb begin
        data_out = reg_file[raddr];
    end

endmodule
```

Changes made:
1. Reduced the address bit-width to 3 bits (`[2:0]`).
2. Changed the memory depth of `reg_file` to 8.
3. Updated the reset loop to iterate for 8 cycles corresponding to the depth of the memory.

### User:
Change the memory depth to 4.

### ChatGPT:
Certainly. Here's the RTL code for the Bias Register with a memory depth of 4:

```systemverilog
module bias_register (
    input logic clk,
    input logic rst_n,
    input logic [1:0] raddr,   // Adjusted for 4 addresses (2 bits)
    input logic [1:0] waddr,   // Adjusted for 4 addresses (2 bits)
    input logic wr_en,
    input logic [31:0] data_in,
    output logic [31:0] data_out
);

    // Register file declaration
    logic [31:0] reg_file [0:3]; // Memory depth changed to 4

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Resetting the register file to all zeros
            for (int i = 0; i < 4; i = i + 1) begin
                reg_file[i] <= 32'b0;
            end
        end else if (wr_en) begin
            // Writing to the register file
            reg_file[waddr] <= data_in;
        end
    end

    // Combinational block for reading the data out
    always_comb begin
        data_out = reg_file[raddr];
    end

endmodule
```

Changes made:
1. Adjusted the address bit-width to 2 bits (`[1:0]`) since we have 4 addresses.
2. Changed the memory depth of `reg_file` to 4.
3. Updated the reset loop to iterate for 4 cycles corresponding to the depth of the memory.
