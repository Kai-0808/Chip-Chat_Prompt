## 04_Bias addition unit
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Bias Addition Unit:

- Input:
  - clk: 1 bit. Clock signal.
  - rst_n: 1 bit. Asynchronous active low reset signal.
  - Data: 2-D 32x32 array of 32-bit pixels. If the data size is smaller, the remaining parts keep being 0.
  - Length: 5 bits. The length of the data. The data is a square array, so the length is the same for both dimensions.
  - Bias value: 32 bits.

- Output:
  - Data with bias: 2-D 32x32 array of 32-bit pixels.

- Functionality:
Add bias to convolved data. For data with size smaller than 32x32, the remaining parts keep being 0. Doing all operations in one cycle is fine. The output should use the sequential logic.

Separate the combinational and sequential logic block. Give the entire SystemVerilog RTL code without any shortcut.

### ChatGPT:
Certainly! Below is the SystemVerilog design for the described "Bias Addition Unit":

```systemverilog
module BiasAdditionUnit (
    input wire clk,
    input wire rst_n,
    input wire [31:0][31:0][31:0] Data,          // 32x32 array of 32-bit pixels
    input wire [4:0] Length,                     // 5-bit length
    input wire [31:0] BiasValue,
    output reg [31:0][31:0][31:0] DataWithBias   // 32x32 array of 32-bit pixels
);

    // Local Variables for combinational logic
    wire [31:0][31:0][31:0] tempDataWithBias;

    // Combinational Logic Block
    always @(*) begin
        for (int i = 0; i < 32; i = i + 1) begin
            for (int j = 0; j < 32; j = j + 1) begin
                if (i < Length && j < Length) begin
                    // Add bias to the data if within the valid length
                    tempDataWithBias[i][j] = Data[i][j] + BiasValue;
                end else begin
                    // Keep the data as zero if outside the valid length
                    tempDataWithBias[i][j] = 32'b0;
                end
            end
        end
    end

    // Sequential Logic Block
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            DataWithBias <= 32'b0; // Active low reset, so we clear the output
        end else begin
            DataWithBias <= tempDataWithBias;
        end
    end

endmodule
```

This design separates the combinational logic from the sequential logic as you requested. In the combinational block, the bias is added only if the current indices `i` and `j` are within the valid `Length` of the data. For indices outside of this, we keep the data as zero, as specified.

In the sequential logic block, we use the clock `clk` to sample the `tempDataWithBias` results into `DataWithBias`. The reset (`rst_n`) is asynchronous and active low, meaning that if `rst_n` is '0', the `DataWithBias` output is reset immediately, regardless of the clock.

### User:
Write it in a systemverilog style. (use logic)

### ChatGPT:
Certainly! Using the `logic` data type and following the SystemVerilog style, here's the "Bias Addition Unit":

```systemverilog
module BiasAdditionUnit (
    input logic clk,
    input logic rst_n,
    input logic [31:0][31:0][31:0] Data,          // 32x32 array of 32-bit pixels
    input logic [4:0] Length,                     // 5-bit length
    input logic [31:0] BiasValue,
    output logic [31:0][31:0][31:0] DataWithBias  // 32x32 array of 32-bit pixels
);

    // Local Variables for combinational logic
    logic [31:0][31:0][31:0] tempDataWithBias;

    // Combinational Logic Block
    always_comb begin
        for (int i = 0; i < 32; i = i + 1) begin
            for (int j = 0; j < 32; j = j + 1) begin
                if (i < Length && j < Length) begin
                    // Add bias to the data if within the valid length
                    tempDataWithBias[i][j] = Data[i][j] + BiasValue;
                end else begin
                    // Keep the data as zero if outside the valid length
                    tempDataWithBias[i][j] = 32'b0;
                end
            end
        end
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            DataWithBias <= 32'b0; // Active low reset, so we clear the output
        end else begin
            DataWithBias <= tempDataWithBias;
        end
    end

endmodule
```

Here are the changes made:

1. Changed `wire` to `logic` for both inputs and outputs as per SystemVerilog.
2. Replaced `always @(*)` with `always_comb` for the combinational block.
3. Replaced `always @(posedge clk or negedge rst_n)` with `always_ff` for the sequential block. 

This code should be more in line with modern SystemVerilog practices.

### User:
DataWithBias <= 32'b0; assign 32 bit to a 32*32*32 array. 

### ChatGPT:
Apologies for the oversight. You're right. Let's correct this. The reset operation should reset the entire `DataWithBias` array to zeros.

Here's the correction:

```systemverilog
module BiasAdditionUnit (
    input logic clk,
    input logic rst_n,
    input logic [31:0][31:0][31:0] Data,          // 32x32 array of 32-bit pixels
    input logic [4:0] Length,                     // 5-bit length
    input logic [31:0] BiasValue,
    output logic [31:0][31:0][31:0] DataWithBias  // 32x32 array of 32-bit pixels
);

    // Local Variables for combinational logic
    logic [31:0][31:0][31:0] tempDataWithBias;

    // Combinational Logic Block
    always_comb begin
        for (int i = 0; i < 32; i = i + 1) begin
            for (int j = 0; j < 32; j = j + 1) begin
                if (i < Length && j < Length) begin
                    // Add bias to the data if within the valid length
                    tempDataWithBias[i][j] = Data[i][j] + BiasValue;
                end else begin
                    // Keep the data as zero if outside the valid length
                    tempDataWithBias[i][j] = 32'b0;
                end
            end
        end
    end

    // Sequential Logic Block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            for (int x = 0; x < 32; x = x + 1) begin
                for (int y = 0; y < 32; y = y + 1) begin
                    DataWithBias[x][y] <= 32'b0;
                end
            end
        end else begin
            DataWithBias <= tempDataWithBias;
        end
    end

endmodule
```

In the reset condition, I've added nested loops to reset the entire `DataWithBias` array to zeros.
