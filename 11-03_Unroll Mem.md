## 11-03_Unroll Mem
### User:
Act as a professional SystemVerilog programmer.
Unroll the for-loops of the following code. You can make shorten the code as long as the logic is the same, e.g., change if-else to `:?` operator, and change always block to assign statement.

Due to the limited text space, you can give some examples of these code without shortcut.

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

### ChatGPT:
Alright, I'll start by unrolling the loops and then provide some examples without shortcuts. Let's begin by addressing the `foreach` loop in the sequential logic block. This loop is used to reset the memory bank. Since we have a 32x32x32 memory bank, the loop will need to be expanded 32 times.

Here's the unrolled code for the memory bank reset:

```systemverilog
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        mem_bank[0] = '{default: '{default: 32'b0}};
        mem_bank[1] = '{default: '{default: 32'b0}};
        mem_bank[2] = '{default: '{default: 32'b0}};
        // ...
        // Continue up to mem_bank[31]
        mem_bank[31] = '{default: '{default: 32'b0}};
    end else begin
        case (wr_en)
            2'b10: mem_bank[waddr] = data_in; // Write the whole image
            2'b11: mem_bank[waddr][row_addr][col_addr] = data_in[row_addr][col_addr]; // Write the specified pixel
            default: ; // No write operation
        endcase
    end
end
```

The `always_comb` and `assign` blocks remain unchanged as there are no loops to unroll there. 

Now, let's provide an example without using shortcuts:

1. Replace `always_comb` with an `always @(*)` block:

```systemverilog
always @(*) begin
    tmp_data_out = mem_bank[raddr];
end
```

2. Replace the `?:` operator in the write logic (if there were any) with a traditional `if-else` structure:

For this specific code, there's no `?:` operator present in the write logic, so we can skip this step.

The other sections of the code remain unchanged.

Would you like any other modifications or further explanations on any section of the code?

### User:
Give a c++ code that can generate these sv code in a file, `unrolled.sv`.

### ChatGPT:
Sure! Below is a C++ program that generates the unrolled SystemVerilog code and saves it to a file named `unrolled.sv`.

```cpp
#include <iostream>
#include <fstream>

int main() {
    std::ofstream svFile;
    svFile.open("unrolled.sv");

    // Write the module header and other static parts
    svFile << "module memory_bank (\n"
              "    input logic clk,\n"
              "    input logic rst_n,\n"
              "    input logic [4:0] raddr,\n"
              "    input logic [4:0] waddr,\n"
              "    input logic [1:0] wr_en,\n"
              "    input logic [4:0] row_addr,\n"
              "    input logic [4:0] col_addr,\n"
              "    input logic [31:0][31:0][31:0] data_in, // 32x32x32 image\n"
              "    output logic [31:0][31:0][31:0] data_out // 32x32x32 image\n"
              ");\n\n"
              "    // Define the memory bank\n"
              "    logic [31:0][31:0][31:0] mem_bank [0:31]; // 32 batches of 32x32x32 images\n\n"
              "    // Signals for intermediate data storage\n"
              "    logic [31:0][31:0][31:0] tmp_data_out;\n\n"
              "    // Combinational Logic Block\n"
              "    always @(*) begin\n"
              "        tmp_data_out = mem_bank[raddr];\n"
              "    end\n\n";

    // Write the always block with unrolled reset logic
    svFile << "    always_ff @(posedge clk or negedge rst_n) begin\n"
              "        if (!rst_n) begin\n";
    for (int i = 0; i < 32; i++) {
        svFile << "            mem_bank[" << i << "] = '{default: '{default: 32'b0}};\n";
    }
    svFile << "        end else begin\n"
              "            case (wr_en)\n"
              "                2'b10: mem_bank[waddr] = data_in; // Write the whole image\n"
              "                2'b11: mem_bank[waddr][row_addr][col_addr] = data_in[row_addr][col_addr]; // Write the specified pixel\n"
              "                default: ; // No write operation\n"
              "            endcase\n"
              "        end\n"
              "    end\n\n";

    // Write the assign statement and close the module
    svFile << "    // Assign output\n"
              "    assign data_out = tmp_data_out;\n\n"
              "endmodule\n";

    svFile.close();

    std::cout << "Generated unrolled.sv successfully!" << std::endl;
    return 0;
}
```

Compile and run this program, and it will generate a file named `unrolled.sv` with the unrolled SystemVerilog code.
