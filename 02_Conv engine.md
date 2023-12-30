## 02_Conv engine
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Convolution Engine:

- Input:
  - clk: 1 bit. Clock signal.
  - rst_n: 1 bit. Asynchronous active low reset signal.
  - start: 1 bit. Synchronous start signal. When start is high, the convolution operation starts.
  - FeatureMap: Full image of size 32x32. Each pixel is 32 bits. If the image size is smaller, the remaining parts keep being 0.
  - Weight: Full kernel of size 7x7. Each pixel is 32 bits. If the kernel size is smaller, the remaining parts keep being 0.
  - MapSize: 5 bits. Specifies the size of the output image ((MapSize+1) x (MapSize+1)).
  - KernelSize: 2 bits. Specifies the size of the convolution kernel. 00: 1x1, 01: 3x3, 10: 5x5, 11: 7x7.
  - RowAddr: 5 bits. The middle row address of the current filtering region.
  - ColAddr: 5 bits. The middle col address of the current filtering region.

- Output:
  - ConvOut: Full image of the convolutional output, where the size is 32x32. Each pixel is 32 bits. If the image size is smaller, the remaining parts keep being 0. They should take values from the register file.
  - done: 1 bit. Done signal. When the convolution operation is finished, the done signal is high.

- Functionality:
Perform convolution operation. There is a 32x32 32-bit register file to store the convolution results. The register file is reset to 0 when the `rst_n` signal is low or the `start` signal is high. The register file updates its values every cycle when performing the convolution operation, and sends the values to the `ConvOut` output. All operations are in raster scan order. Zero padding are used for the input image.

There are two inner counters to indicate the address (row and col) of output pixels. When `start` is high, reset the counters and register file. At each cycle, perform the convolution operation on the current pixel based on `RowAddr` and `ColAddr`, update the counters, and send the result to the register file based on the current counter values. When the counters reach the end of the output image, set the `done` signal high.

Separate the combinational and sequential logic block. Give the entire SystemVerilog RTL code without any shortcut.

### ChatGPT:
Sure, here's a SystemVerilog module that implements the described convolution engine:

```systemverilog
module ConvolutionEngine (
    input logic clk,
    input logic rst_n,
    input logic start,
    input logic [31:0][31:0][31:0] FeatureMap,
    input logic [6:0][6:0][31:0] Weight,
    input logic [4:0] MapSize,
    input logic [1:0] KernelSize,
    input logic [4:0] RowAddr,
    input logic [4:0] ColAddr,
    output logic [31:0][31:0][31:0] ConvOut,
    output logic done
);

    // Define the size of the kernel based on KernelSize
    parameter [1:0] SIZE_1x1 = 2'b00;
    parameter [1:0] SIZE_3x3 = 2'b01;
    parameter [1:0] SIZE_5x5 = 2'b10;
    parameter [1:0] SIZE_7x7 = 2'b11;

    // Internal counters
    logic [4:0] row_counter;
    logic [4:0] col_counter;

    // Register file
    logic [31:0][31:0][31:0] reg_file;

    // Combinational block for convolution operation
    always_comb begin
        // ... Perform convolution operation based on RowAddr, ColAddr, FeatureMap, and Weight ...
        // ... Store the result in a temporary variable ...
    end

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n || start) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else begin
            // ... Update reg_file with the result from the combinational block ...
            // ... Update row_counter and col_counter ...
            // ... Set done signal when counters reach the end ...
        end
    end

    // Output assignment
    assign ConvOut = reg_file;

endmodule
```

Note: This is a high-level representation of the module. The actual convolution operation, counter updates, and other details need to be filled in. The code is structured to separate the combinational and sequential logic as requested.

### User:
Complete all the shortcut parts `// ...`.

### ChatGPT:
Certainly! Here's the completed SystemVerilog module for the convolution engine:

```systemverilog
module ConvolutionEngine (
    input logic clk,
    input logic rst_n,
    input logic start,
    input logic [31:0][31:0][31:0] FeatureMap,
    input logic [6:0][6:0][31:0] Weight,
    input logic [4:0] MapSize,
    input logic [1:0] KernelSize,
    input logic [4:0] RowAddr,
    input logic [4:0] ColAddr,
    output logic [31:0][31:0][31:0] ConvOut,
    output logic done
);

    // Define the size of the kernel based on KernelSize
    parameter [1:0] SIZE_1x1 = 2'b00;
    parameter [1:0] SIZE_3x3 = 2'b01;
    parameter [1:0] SIZE_5x5 = 2'b10;
    parameter [1:0] SIZE_7x7 = 2'b11;

    // Internal counters
    logic [4:0] row_counter;
    logic [4:0] col_counter;

    // Register file
    logic [31:0][31:0][31:0] reg_file;
    logic [31:0] temp_result;

    // Combinational block for convolution operation
    always_comb begin
        integer i, j;
        integer kernel_offset;
        temp_result = 0;

        case(KernelSize)
            SIZE_1x1: kernel_offset = 0;
            SIZE_3x3: kernel_offset = 1;
            SIZE_5x5: kernel_offset = 2;
            SIZE_7x7: kernel_offset = 3;
        endcase

        for (i = -kernel_offset; i <= kernel_offset; i++) begin
            for (j = -kernel_offset; j <= kernel_offset; j++) begin
                temp_result = temp_result + FeatureMap[RowAddr + i][ColAddr + j] * Weight[i + kernel_offset][j + kernel_offset];
            end
        end
    end

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n || start) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else begin
            reg_file[row_counter][col_counter] <= temp_result;
            if (col_counter == MapSize) begin
                col_counter <= '0;
                if (row_counter == MapSize) begin
                    done <= 1;
                end else begin
                    row_counter <= row_counter + 1;
                end
            end else begin
                col_counter <= col_counter + 1;
            end
        end
    end

    // Output assignment
    assign ConvOut = reg_file;

endmodule
```

This module now performs the convolution operation based on the given `RowAddr` and `ColAddr`, updates the counters, and sets the `done` signal when the counters reach the end of the output image. The convolution operation is performed in raster scan order with zero padding for the input image.

### User:
For the `always_comb` block, move the generate variables outside the block. Use the fix range of for loop, e.g., `for(i = -3; i <= 3; i++)`, and then use condition to determine whether the data is valid based on the `KernelSize` and zero padding.

### ChatGPT:
Sure! Here's the modified `always_comb` block with the generate variables moved outside and the fixed range for the loop:

```systemverilog
module ConvolutionEngine (
    // ... [same as before]
);

    // ... [same as before]

    // Generate variables
    integer i, j;
    logic [31:0] temp_pixel;
    logic [31:0] temp_weight;

    // Combinational block for convolution operation
    always_comb begin
        temp_result = 0;

        for (i = -3; i <= 3; i++) begin
            for (j = -3; j <= 3; j++) begin
                // Check if the current position is within the valid kernel size
                if ((KernelSize == SIZE_1x1 && i == 0 && j == 0) ||
                    (KernelSize == SIZE_3x3 && i >= -1 && i <= 1 && j >= -1 && j <= 1) ||
                    (KernelSize == SIZE_5x5 && i >= -2 && i <= 2 && j >= -2 && j <= 2) ||
                    (KernelSize == SIZE_7x7)) begin

                    // Zero padding for the input image
                    if (RowAddr + i < 0 || RowAddr + i > 31 || ColAddr + j < 0 || ColAddr + j > 31) begin
                        temp_pixel = 0;
                    end else begin
                        temp_pixel = FeatureMap[RowAddr + i][ColAddr + j];
                    end

                    temp_weight = Weight[i + 3][j + 3];
                    temp_result = temp_result + temp_pixel * temp_weight;
                end
            end
        end
    end

    // ... [same as before]

endmodule
```

The loop now iterates over the fixed range from `-3` to `3`, and the conditions inside the loop determine whether the current position is within the valid kernel size. The zero padding for the input image is also handled inside the loop.

### User:
The `temp_weight` may vary from `KernelSize`. Also, `temp_pixel` and `temp_weight` need to be 7x7 array ([6:0][6:0][31:0]) to avoid overwrite and overdrive, where in each statement temp[i+3][j+3] is used.

### ChatGPT:
Understood. I'll make the necessary modifications. Here's the updated code:

```systemverilog
module ConvolutionEngine (
    // ... [same as before]
);

    // ... [same as before]

    // Generate variables
    integer i, j;
    logic [6:0][6:0][31:0] temp_pixel;
    logic [6:0][6:0][31:0] temp_weight;

    // Combinational block for convolution operation
    always_comb begin
        temp_result = 0;

        for (i = -3; i <= 3; i++) begin
            for (j = -3; j <= 3; j++) begin
                // Zero padding for the input image
                if (RowAddr + i < 0 || RowAddr + i > 31 || ColAddr + j < 0 || ColAddr + j > 31) begin
                    temp_pixel[i+3][j+3] = 0;
                end else begin
                    temp_pixel[i+3][j+3] = FeatureMap[RowAddr + i][ColAddr + j];
                end

                // Assign weights based on KernelSize
                if ((KernelSize == SIZE_1x1 && i == 0 && j == 0) ||
                    (KernelSize == SIZE_3x3 && i >= -1 && i <= 1 && j >= -1 && j <= 1) ||
                    (KernelSize == SIZE_5x5 && i >= -2 && i <= 2 && j >= -2 && j <= 2) ||
                    (KernelSize == SIZE_7x7)) begin

                    temp_weight[i+3][j+3] = Weight[i+3][j+3];
                end else begin
                    temp_weight[i+3][j+3] = 0;
                end

                temp_result = temp_result + temp_pixel[i+3][j+3] * temp_weight[i+3][j+3];
            end
        end
    end

    // ... [same as before]

endmodule
```

I've now made `temp_pixel` and `temp_weight` as 7x7 arrays. The weights are assigned based on the `KernelSize`, and the convolution operation is performed using these arrays.

### User:
The `temp_weight` may vary from `KernelSize`.
For example, when `SIZE_1x1`, it should be Weight[i][j].

### ChatGPT:
Got it. I'll modify the assignment of `temp_weight` to consider the `KernelSize` and the current loop indices `i` and `j`. Here's the updated code:

```systemverilog
module ConvolutionEngine (
    // ... [same as before]
);

    // ... [same as before]

    // Generate variables
    integer i, j;
    logic [6:0][6:0][31:0] temp_pixel;
    logic [6:0][6:0][31:0] temp_weight;

    // Combinational block for convolution operation
    always_comb begin
        temp_result = 0;

        for (i = -3; i <= 3; i++) begin
            for (j = -3; j <= 3; j++) begin
                // Zero padding for the input image
                if (RowAddr + i < 0 || RowAddr + i > 31 || ColAddr + j < 0 || ColAddr + j > 31) begin
                    temp_pixel[i+3][j+3] = 0;
                end else begin
                    temp_pixel[i+3][j+3] = FeatureMap[RowAddr + i][ColAddr + j];
                end

                // Assign weights based on KernelSize
                if (KernelSize == SIZE_1x1) begin
                    temp_weight[i+3][j+3] = (i == 0 && j == 0) ? Weight[0][0] : 0;
                end else if (KernelSize == SIZE_3x3) begin
                    temp_weight[i+3][j+3] = (i >= -1 && i <= 1 && j >= -1 && j <= 1) ? Weight[i+1][j+1] : 0;
                end else if (KernelSize == SIZE_5x5) begin
                    temp_weight[i+3][j+3] = (i >= -2 && i <= 2 && j >= -2 && j <= 2) ? Weight[i+2][j+2] : 0;
                end else if (KernelSize == SIZE_7x7) begin
                    temp_weight[i+3][j+3] = Weight[i+3][j+3];
                end else begin
                    temp_weight[i+3][j+3] = 0;
                end

                temp_result = temp_result + temp_pixel[i+3][j+3] * temp_weight[i+3][j+3];
            end
        end
    end

    // ... [same as before]

endmodule
```

Now, the assignment of `temp_weight` considers the `KernelSize` and the current loop indices `i` and `j` to fetch the appropriate weight value.

### User:
Great! 
The part `!rst_n || start` would cause synthesis error. Make `start` condition be synchronous to do the reset.

### ChatGPT:
Understood. I'll modify the reset condition to make the `start` condition synchronous. Here's the updated code:

```systemverilog
module ConvolutionEngine (
    // ... [same as before]
);

    // ... [same as before]

    // Generate variables
    integer i, j;
    logic [6:0][6:0][31:0] temp_pixel;
    logic [6:0][6:0][31:0] temp_weight;

    // Combinational block for convolution operation
    // ... [same as before]

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else if (start) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else begin
            // ... [same as before]
        end
    end

    // ... [same as before]

endmodule
```

Now, the `start` condition is synchronous and will reset the module only on the rising edge of the `clk` signal when `start` is high.

### User:
Show the entire code.

### ChatGPT:
Of course! Here's the entire SystemVerilog module for the convolution engine:

```systemverilog
module ConvolutionEngine (
    input logic clk,
    input logic rst_n,
    input logic start,
    input logic [31:0][31:0][31:0] FeatureMap,
    input logic [6:0][6:0][31:0] Weight,
    input logic [4:0] MapSize,
    input logic [1:0] KernelSize,
    input logic [4:0] RowAddr,
    input logic [4:0] ColAddr,
    output logic [31:0][31:0][31:0] ConvOut,
    output logic done
);

    // Define the size of the kernel based on KernelSize
    parameter [1:0] SIZE_1x1 = 2'b00;
    parameter [1:0] SIZE_3x3 = 2'b01;
    parameter [1:0] SIZE_5x5 = 2'b10;
    parameter [1:0] SIZE_7x7 = 2'b11;

    // Internal counters
    logic [4:0] row_counter;
    logic [4:0] col_counter;

    // Register file
    logic [31:0][31:0][31:0] reg_file;
    logic [31:0] temp_result;

    // Generate variables
    integer i, j;
    logic [6:0][6:0][31:0] temp_pixel;
    logic [6:0][6:0][31:0] temp_weight;

    // Combinational block for convolution operation
    always_comb begin
        temp_result = 0;

        for (i = -3; i <= 3; i++) begin
            for (j = -3; j <= 3; j++) begin
                // Zero padding for the input image
                if (RowAddr + i < 0 || RowAddr + i > 31 || ColAddr + j < 0 || ColAddr + j > 31) begin
                    temp_pixel[i+3][j+3] = 0;
                end else begin
                    temp_pixel[i+3][j+3] = FeatureMap[RowAddr + i][ColAddr + j];
                end

                // Assign weights based on KernelSize
                if (KernelSize == SIZE_1x1) begin
                    temp_weight[i+3][j+3] = (i == 0 && j == 0) ? Weight[0][0] : 0;
                end else if (KernelSize == SIZE_3x3) begin
                    temp_weight[i+3][j+3] = (i >= -1 && i <= 1 && j >= -1 && j <= 1) ? Weight[i+1][j+1] : 0;
                end else if (KernelSize == SIZE_5x5) begin
                    temp_weight[i+3][j+3] = (i >= -2 && i <= 2 && j >= -2 && j <= 2) ? Weight[i+2][j+2] : 0;
                end else if (KernelSize == SIZE_7x7) begin
                    temp_weight[i+3][j+3] = Weight[i+3][j+3];
                end else begin
                    temp_weight[i+3][j+3] = 0;
                end

                temp_result = temp_result + temp_pixel[i+3][j+3] * temp_weight[i+3][j+3];
            end
        end
    end

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else if (start) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else begin
            reg_file[row_counter][col_counter] <= temp_result;
            if (col_counter == MapSize) begin
                col_counter <= '0;
                if (row_counter == MapSize) begin
                    done <= 1;
                end else begin
                    row_counter <= row_counter + 1;
                end
            end else begin
                col_counter <= col_counter + 1;
            end
        end
    end

    // Output assignment
    assign ConvOut = reg_file;

endmodule
```

This module now performs the convolution operation based on the given `RowAddr` and `ColAddr`, updates the counters, and sets the `done` signal when the counters reach the end of the output image. The convolution operation is performed in raster scan order with zero padding for the input image.

### User:
For the counters control, change the conditions to `if(col/row_counter == MapSize -1)`.

### ChatGPT:
Understood. I'll modify the counter control logic accordingly. Here's the updated code for the sequential block:

```systemverilog
// Sequential block
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        reg_file <= '0;
        row_counter <= '0;
        col_counter <= '0;
        done <= 0;
    end else if (start) begin
        reg_file <= '0;
        row_counter <= '0;
        col_counter <= '0;
        done <= 0;
    end else begin
        reg_file[row_counter][col_counter] <= temp_result;
        if (col_counter == MapSize - 1) begin
            col_counter <= '0;
            if (row_counter == MapSize - 1) begin
                done <= 1;
            end else begin
                row_counter <= row_counter + 1;
            end
        end else begin
            col_counter <= col_counter + 1;
        end
    end
end
```

I've updated the conditions for the counters to check if they are equal to `MapSize - 1` as you requested.

### User:
Give the entire code.

### ChatGPT:
Certainly! Here's the entire SystemVerilog module for the convolution engine with the updated counter control logic:

```systemverilog
module ConvolutionEngine (
    input logic clk,
    input logic rst_n,
    input logic start,
    input logic [31:0][31:0][31:0] FeatureMap,
    input logic [6:0][6:0][31:0] Weight,
    input logic [4:0] MapSize,
    input logic [1:0] KernelSize,
    input logic [4:0] RowAddr,
    input logic [4:0] ColAddr,
    output logic [31:0][31:0][31:0] ConvOut,
    output logic done
);

    // Define the size of the kernel based on KernelSize
    parameter [1:0] SIZE_1x1 = 2'b00;
    parameter [1:0] SIZE_3x3 = 2'b01;
    parameter [1:0] SIZE_5x5 = 2'b10;
    parameter [1:0] SIZE_7x7 = 2'b11;

    // Internal counters
    logic [4:0] row_counter;
    logic [4:0] col_counter;

    // Register file
    logic [31:0][31:0][31:0] reg_file;
    logic [31:0] temp_result;

    // Generate variables
    integer i, j;
    logic [6:0][6:0][31:0] temp_pixel;
    logic [6:0][6:0][31:0] temp_weight;

    // Combinational block for convolution operation
    always_comb begin
        temp_result = 0;

        for (i = -3; i <= 3; i++) begin
            for (j = -3; j <= 3; j++) begin
                // Zero padding for the input image
                if (RowAddr + i < 0 || RowAddr + i > 31 || ColAddr + j < 0 || ColAddr + j > 31) begin
                    temp_pixel[i+3][j+3] = 0;
                end else begin
                    temp_pixel[i+3][j+3] = FeatureMap[RowAddr + i][ColAddr + j];
                end

                // Assign weights based on KernelSize
                if (KernelSize == SIZE_1x1) begin
                    temp_weight[i+3][j+3] = (i == 0 && j == 0) ? Weight[0][0] : 0;
                end else if (KernelSize == SIZE_3x3) begin
                    temp_weight[i+3][j+3] = (i >= -1 && i <= 1 && j >= -1 && j <= 1) ? Weight[i+1][j+1] : 0;
                end else if (KernelSize == SIZE_5x5) begin
                    temp_weight[i+3][j+3] = (i >= -2 && i <= 2 && j >= -2 && j <= 2) ? Weight[i+2][j+2] : 0;
                end else if (KernelSize == SIZE_7x7) begin
                    temp_weight[i+3][j+3] = Weight[i+3][j+3];
                end else begin
                    temp_weight[i+3][j+3] = 0;
                end

                temp_result = temp_result + temp_pixel[i+3][j+3] * temp_weight[i+3][j+3];
            end
        end
    end

    // Sequential block
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else if (start) begin
            reg_file <= '0;
            row_counter <= '0;
            col_counter <= '0;
            done <= 0;
        end else begin
            reg_file[row_counter][col_counter] <= temp_result;
            if (col_counter == MapSize - 1) begin
                col_counter <= '0;
                if (row_counter == MapSize - 1) begin
                    done <= 1;
                end else begin
                    row_counter <= row_counter + 1;
                end
            end else begin
                col_counter <= col_counter + 1;
            end
        end
    end

    // Output assignment
    assign ConvOut = reg_file;

endmodule
```

This module now performs the convolution operation based on the given `RowAddr` and `ColAddr`, updates the counters using the modified conditions, and sets the `done` signal when the counters reach the end of the output image. The convolution operation is performed in raster scan order with zero padding for the input image.
