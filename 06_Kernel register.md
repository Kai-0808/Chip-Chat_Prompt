## 06_Kernel register
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Kernel Register:

- Input:
  - clk: 1 bit. Clock signal.
  - rst_n: 1 bit. Asynchronous active low reset signal.
  - raddr: 5 bits. The address of the data to be read.
  - waddr: 5 bits. The address of the data to be written.
  - wr_en: 1 bit. Write enable signal. 0: inactive, 1: write the single pixel.
  - row_addr: 5 bits. The row address of the data to be written. Only used when wr_en is 11.
  - col_addr: 5 bits. The col address of the data to be written. Only used when wr_en is 11.
  - data_in: 32 bits. The data to be written.

- Output:
  - data_out: 2-D 7x7 image of 32-bit pixels, i.e., `logic [6:0][6:0][31:0] data_out`. The data read from the register file.

- Functionality:
The register file has batch (x32) of 7x7 image of 32-bit pixel, i.e., `logic [6:0][6:0][31:0] reg_file [0:31]`. The register file is reset to 0 when the `rst_n` signal is low.
`raddr/waddr` infers the address of image. The register file updates its values to the address specified by `waddr` every cycle when performing the write operation, and always sends the values to the `data_out` output from the address specified by `raddr`.

When `wr_en` is 0, the register file is inactive.
When `wr_en` is 1, the register file writes the single pixel from `data_in` to the address specified by `row_addr` and `col_addr`, and the remaining pixels keep their original values.

Separate the combinational and sequential logic block. Give the entire SystemVerilog RTL code without any shortcut.

### ChatGPT:
To design the Kernel Register module as described, we will follow the following steps:

1. Define the module and its input-output ports.
2. Create internal storage elements.
3. Separate combinational and sequential logic.

The design will look something like this:

```systemverilog
module KernelRegister(
    input logic clk,      // Clock
    input logic rst_n,    // Asynchronous Reset
    input logic [4:0] raddr, waddr,  // Read and Write Addresses for the batch
    input logic wr_en,                // Write Enable
    input logic [4:0] row_addr, col_addr, // Pixel addresses
    input logic [31:0] data_in,       // Data to write
    output logic [6:0][6:0][31:0] data_out  // Data to read out
);

    // Register file declaration
    logic [6:0][6:0][31:0] reg_file [0:31]; // 32 batches of 7x7 images

    // Combinational Logic Block
    always_comb begin
        // Default data_out value
        data_out = reg_file[raddr];

        if (wr_en) begin
            // When wr_en is active, modify the selected pixel and leave the rest intact
            data_out[row_addr][col_addr] = data_in;
        end
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Asynchronous reset
            reg_file <= '0; // Reset all registers
        end else if (wr_en) begin
            // Update only the selected pixel in the selected batch
            reg_file[waddr][row_addr][col_addr] <= data_in;
        end
    end

endmodule
```

Note:

1. This module doesn't infer memory primitives found in FPGAs or ASICs; instead, it uses 2-D registers. If targeting specific hardware, a different approach using BRAMs or other memory structures might be more appropriate.
2. The design provided is synchronous to `clk` and respects the asynchronous active-low reset.
3. The combinational logic block updates `data_out` based on `raddr`, and potentially modifies the pixel based on `wr_en`, `row_addr`, and `col_addr`.
4. The sequential logic block updates the `reg_file` if the `wr_en` is active.
5. Proper care should be taken during synthesis and place and route to ensure performance and timing constraints are met.

### User:
Always let `data_out = reg_file[raddr]`.
Also, use the fomat of `foreach(reg_file[i]) reg_file[i] = '{default: '{default: 32'b0}};` to reset the register file.

### ChatGPT:
Certainly! I've updated the module with the mentioned corrections:

1. The `data_out` will always be `reg_file[raddr]` regardless of the `wr_en` value.
2. Used the recommended format for resetting the `reg_file` using `foreach`.

Here's the updated SystemVerilog RTL:

```systemverilog
module KernelRegister(
    input logic clk,      // Clock
    input logic rst_n,    // Asynchronous Reset
    input logic [4:0] raddr, waddr,  // Read and Write Addresses for the batch
    input logic wr_en,                // Write Enable
    input logic [4:0] row_addr, col_addr, // Pixel addresses
    input logic [31:0] data_in,       // Data to write
    output logic [6:0][6:0][31:0] data_out  // Data to read out
);

    // Register file declaration
    logic [6:0][6:0][31:0] reg_file [0:31]; // 32 batches of 7x7 images

    // Combinational Logic Block for data_out
    always_comb begin
        data_out = reg_file[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Asynchronous reset using foreach
            foreach(reg_file[i]) reg_file[i] = '{default: '{default: 32'b0}};
        end else if (wr_en) begin
            // Update only the selected pixel in the selected batch
            reg_file[waddr][row_addr][col_addr] <= data_in;
        end
    end

endmodule
```

With this modification, the `data_out` output always reflects the image in the `reg_file` as specified by `raddr`, and the reset structure uses the recommended `foreach` method.

### User:
Reduce the addr to 3-bit and change the memory depth to 8.

### ChatGPT:
Got it. I've made the adjustments:

- Changed the address (`raddr`, `waddr`) width to 3 bits.
- Changed the depth of the `reg_file` to 8.

Here's the updated SystemVerilog RTL:

```systemverilog
module KernelRegister(
    input logic clk,      // Clock
    input logic rst_n,    // Asynchronous Reset
    input logic [2:0] raddr, waddr,  // Read and Write Addresses for the batch
    input logic wr_en,                // Write Enable
    input logic [4:0] row_addr, col_addr, // Pixel addresses
    input logic [31:0] data_in,       // Data to write
    output logic [6:0][6:0][31:0] data_out  // Data to read out
);

    // Adjusted Register file declaration for depth of 8
    logic [6:0][6:0][31:0] reg_file [0:7]; // 8 batches of 7x7 images

    // Combinational Logic Block for data_out
    always_comb begin
        data_out = reg_file[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Asynchronous reset using foreach
            foreach(reg_file[i]) reg_file[i] = '{default: '{default: 32'b0}};
        end else if (wr_en) begin
            // Update only the selected pixel in the selected batch
            reg_file[waddr][row_addr][col_addr] <= data_in;
        end
    end

endmodule
```

These modifications reflect the desired changes in address width and memory depth.

### User:
Change the memory depth to 4.

### ChatGPT:
Alright, I'll reduce the depth of `reg_file` to 4 and adjust the address (`raddr`, `waddr`) width to 2 bits accordingly. Here's the updated SystemVerilog RTL:

```systemverilog
module KernelRegister(
    input logic clk,      // Clock
    input logic rst_n,    // Asynchronous Reset
    input logic [1:0] raddr, waddr,  // Read and Write Addresses for the batch
    input logic wr_en,                // Write Enable
    input logic [4:0] row_addr, col_addr, // Pixel addresses
    input logic [31:0] data_in,       // Data to write
    output logic [6:0][6:0][31:0] data_out  // Data to read out
);

    // Adjusted Register file declaration for depth of 4
    logic [6:0][6:0][31:0] reg_file [0:3]; // 4 batches of 7x7 images

    // Combinational Logic Block for data_out
    always_comb begin
        data_out = reg_file[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            // Asynchronous reset using foreach
            foreach(reg_file[i]) reg_file[i] = '{default: '{default: 32'b0}};
        end else if (wr_en) begin
            // Update only the selected pixel in the selected batch
            reg_file[waddr][row_addr][col_addr] <= data_in;
        end
    end

endmodule
```

Now, the memory depth is 4, and the address lines are 2 bits wide.
