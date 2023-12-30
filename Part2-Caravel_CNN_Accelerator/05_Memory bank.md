## 05_Memory bank
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Memory Bank:

- Input:
  - clk: 1 bit. Clock signal.
  - rst_n: 1 bit. Asynchronous active low reset signal.
  - raddr: 5 bits. The address of the data to be read.
  - waddr: 5 bits. The address of the data to be written.
  - wr_en: 2 bit. Write enable signal. 00/01: inactive, 10: write the whole image, 11: write the single pixel.
  - row_addr: 5 bits. The row address of the data to be written. Only used when wr_en is 11.
  - col_addr: 5 bits. The col address of the data to be written. Only used when wr_en is 11.
  - data_in: 2-D 32x32 image of 32-bit pixels, i.e., the total size is 32x32x32 bits. The data to be written.

- Output:
  - data_out: 2-D 32x32 image of 32-bit pixels, i.e., the total size is 32x32x32 bits. The data read from the memory.

- Functionality:
The memory bank has batch (x32) of 32x32 image of 32-bit pixel, which means the total size is 32x32x32x32 bits. The memory bank is reset to 0 when the `rst_n` signal is low.
`raddr/waddr` infers the address of image. The memory bank updates its values to the address specified by `waddr` every cycle when performing the write operation, and always sends the values to the `data_out` output from the address specified by `raddr`.

When `wr_en` is 00 or 01, the memory bank is inactive.
When `wr_en` is 10, the memory bank writes the whole image from `data_in`.
When `wr_en` is 11, the memory bank writes the single pixel from `data_in` to the address specified by `row_addr` and `col_addr`, and the remaining pixels keep their original values.

Separate the combinational and sequential logic block. Give the entire SystemVerilog RTL code without any shortcut.

### ChatGPT:
Certainly! Below is the SystemVerilog RTL design based on the provided specification:

```systemverilog
module memory_bank (
    input logic clk,
    input logic rst_n,
    input logic [4:0] raddr,
    input logic [4:0] waddr,
    input logic [1:0] wr_en,
    input logic [4:0] row_addr,
    input logic [4:0] col_addr,
    input logic [31:0][31:0] data_in, // 32x32 image of 32-bit pixels
    output logic [31:0][31:0] data_out // 32x32 image of 32-bit pixels
);

    // Define the memory bank
    logic [31:0][31:0][31:0][4:0] mem_bank; // 32x32x32x32 memory bank

    // Signals for intermediate data storage
    logic [31:0][31:0] tmp_data_out;

    // Combinational Logic Block
    always_comb begin
        tmp_data_out = mem_bank[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            mem_bank <= '{default: '{default: '{default: 32'b0}}}; // Reset the memory bank
        end else begin
            case (wr_en)
                2'b10: mem_bank[waddr] <= data_in; // Write the whole image
                2'b11: begin
                    // Write only the specific pixel
                    mem_bank[waddr][row_addr][col_addr] <= data_in[row_addr][col_addr];
                end
                default: ; // No write operation
            endcase
        end
    end

    // Assign output
    assign data_out = tmp_data_out;

endmodule
```

Explanation:
1. The `mem_bank` variable is a 4D array that represents the memory, storing a batch of 32 images where each image is a 32x32 pixel and each pixel is 32 bits.
2. The `tmp_data_out` signal is an intermediate variable to hold the data read from memory in the combinational block.
3. In the combinational logic block, the value from the memory at address `raddr` is assigned to `tmp_data_out`.
4. The sequential block checks for reset first. If reset is asserted, it clears the memory bank. Otherwise, based on the `wr_en` value, it either writes the whole image or a single pixel to the specified address in the memory bank.
5. The `data_out` is assigned the value from `tmp_data_out`.

Note: This module assumes that only one of the write operations is active at any given time. If both are active, the behavior might be undefined. Proper arbitration would be needed in case both write enables can be active simultaneously.

### User:
The image size is wrong!
Use `logic [31:0][31:0][31:0] data_in/data_out` and `logic [31:0][31:0][31:0] mem_bank [0:31]`.

### ChatGPT:
I apologize for the oversight. Let's correct that. Given the updated image dimensions and the description, here's the corrected module:

```systemverilog
module memory_bank (
    input logic clk,
    input logic rst_n,
    input logic [4:0] raddr,
    input logic [4:0] waddr,
    input logic [1:0] wr_en,
    input logic [4:0] row_addr,
    input logic [4:0] col_addr,
    input logic [31:0][31:0][31:0] data_in, // 32x32x32 image
    output logic [31:0][31:0][31:0] data_out // 32x32x32 image
);

    // Define the memory bank
    logic [31:0][31:0][31:0] mem_bank [0:31]; // 32 batches of 32x32x32 images

    // Signals for intermediate data storage
    logic [31:0][31:0][31:0] tmp_data_out;

    // Combinational Logic Block
    always_comb begin
        tmp_data_out = mem_bank[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            foreach (mem_bank[i]) mem_bank[i] = '{default: '{default: 32'b0}}; // Reset the memory bank
        end else begin
            case (wr_en)
                2'b10: mem_bank[waddr] = data_in; // Write the whole image
                2'b11: mem_bank[waddr][row_addr][col_addr] = data_in[row_addr][col_addr]; // Write the specified pixel
                default: ; // No write operation
            endcase
        end
    end

    // Assign output
    assign data_out = tmp_data_out;

endmodule
```

This module now correctly represents a memory bank that can store 32 batches of 32x32x32 images and provides the functionalities described in the specifications.

### User:
Reduce the addr to 3-bit and change the memory depth to 8.

### ChatGPT:
Of course, if we reduce the address size to 3 bits, we have addresses ranging from `000` to `111`, which gives us a total of 8 unique addresses. Here's the modified module with the address size reduced to 3 bits and the memory depth changed to 8:

```systemverilog
module memory_bank (
    input logic clk,
    input logic rst_n,
    input logic [2:0] raddr,       // Reduced to 3 bits
    input logic [2:0] waddr,       // Reduced to 3 bits
    input logic [1:0] wr_en,
    input logic [4:0] row_addr,
    input logic [4:0] col_addr,
    input logic [31:0][31:0][31:0] data_in, // 32x32x32 image
    output logic [31:0][31:0][31:0] data_out // 32x32x32 image
);

    // Define the memory bank
    logic [31:0][31:0][31:0] mem_bank [0:7]; // 8 batches of 32x32x32 images, changed depth to 8

    // Signals for intermediate data storage
    logic [31:0][31:0][31:0] tmp_data_out;

    // Combinational Logic Block
    always_comb begin
        tmp_data_out = mem_bank[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            foreach (mem_bank[i]) mem_bank[i] = '{default: '{default: 32'b0}}; // Reset the memory bank
        end else begin
            case (wr_en)
                2'b10: mem_bank[waddr] = data_in; // Write the whole image
                2'b11: mem_bank[waddr][row_addr][col_addr] = data_in[row_addr][col_addr]; // Write the specified pixel
                default: ; // No write operation
            endcase
        end
    end

    // Assign output
    assign data_out = tmp_data_out;

endmodule
```

With the modifications, the memory bank can now store 8 batches of 32x32x32 images, and the address bus for reading/writing is now 3 bits.

### User:
Change the memory depth to 4.

### ChatGPT:
Sure! If we're reducing the memory depth to 4, then the address size will remain as 2 bits. Here's the module with the memory depth changed to 4:

```systemverilog
module memory_bank (
    input logic clk,
    input logic rst_n,
    input logic [1:0] raddr,       // Reduced to 2 bits for 4 memory slots
    input logic [1:0] waddr,       // Reduced to 2 bits for 4 memory slots
    input logic [1:0] wr_en,
    input logic [4:0] row_addr,
    input logic [4:0] col_addr,
    input logic [31:0][31:0][31:0] data_in, // 32x32x32 image
    output logic [31:0][31:0][31:0] data_out // 32x32x32 image
);

    // Define the memory bank
    logic [31:0][31:0][31:0] mem_bank [0:3]; // 4 batches of 32x32x32 images

    // Signals for intermediate data storage
    logic [31:0][31:0][31:0] tmp_data_out;

    // Combinational Logic Block
    always_comb begin
        tmp_data_out = mem_bank[raddr];
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            foreach (mem_bank[i]) mem_bank[i] = '{default: '{default: 32'b0}}; // Reset the memory bank
        end else begin
            case (wr_en)
                2'b10: mem_bank[waddr] = data_in; // Write the whole image
                2'b11: mem_bank[waddr][row_addr][col_addr] = data_in[row_addr][col_addr]; // Write the specified pixel
                default: ; // No write operation
            endcase
        end
    end

    // Assign output
    assign data_out = tmp_data_out;

endmodule
```

With the updated design, the memory bank can store 4 batches of 32x32x32 images, and the address bus for reading/writing is 2 bits.
