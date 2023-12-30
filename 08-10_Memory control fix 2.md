## 08-10_Memory control fix 2
### User:
For the memory wrapper, there should be a input `logic [4:0] daddr`.
When Conv, Act, Pooling, and Bias, the `waddr` of memory bank should be `daddr`, and when Load, the `waddr` will then be `mem_or_reg_addr`.
Other logic should remain unchanged.

Recall:
```systemverilog
module MemoryWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [31:0] wb_addr,
    input logic [4:0] saddr,
    input logic [4:0] sub_addr,
    input logic done,
    input logic [31:0] data_in, 
    input logic [31:0][31:0][31:0] padded_output, post_act_data, pooled_output_data, data_with_bias,
    output logic [31:0][31:0][31:0] data_from_mem,
    output logic [6:0][6:0][31:0] kernel_data,
    output logic [31:0] bias_data_from_reg
);

    // Signals for memory_bank
    logic [31:0][31:0][31:0] mem_data_in;
    logic [1:0] wr_en_signal;

    // Signals for KernelRegister
    logic [31:0] kernel_data_in;

    // Signals for Bias_Register
    logic [31:0] bias_data_in;

    // Extracted addresses from wb_addr
    logic [4:0] mem_row_addr, mem_col_addr, mem_or_reg_addr;
    logic [1:0] mem_type_from_wb_addr;

    assign mem_row_addr = wb_addr[11:7];
    assign mem_col_addr = wb_addr[6:2];
    assign mem_or_reg_addr = wb_addr[16:12];
    assign mem_type_from_wb_addr = wb_addr[18:17];

    // Logic for setting addresses, controlling writes, and feeding data
    always_comb begin
        // Default values
        mem_data_in = 32'b0;
        kernel_data_in = 32'b0;
        bias_data_in = 32'b0;
        wr_en_signal = 2'b00; // No write by default for memory bank
        case (opcode)
            3'b001: begin // Conv
                if (done) begin
                    mem_data_in = padded_output;
                    wr_en_signal = 2'b10;       // Write enable for memory bank
                end
            end
            3'b010: begin // Activation
                if (done) begin
                    mem_data_in = post_act_data; // Assuming post_act_data holds the result from ActivationFunctionUnit
                    wr_en_signal = 2'b10;        // Write enable for memory bank
                end
            end
            3'b011: begin // Pooling
                if (done) begin
                    mem_data_in = pooled_output_data; 
                    wr_en_signal = 2'b10;      // Write enable for memory bank
                end
            end
            3'b100: begin // Bias
                if (done) begin
                    mem_data_in = data_with_bias; // Assuming data_with_bias holds the result from BiasAdditionUnit
                    wr_en_signal = 2'b10;         // Write enable for memory bank
                end
            end
            3'b101: begin // Load
                case (mem_type_from_wb_addr)
                    2'b00: begin // Memory bank
                        mem_data_in[mem_row_addr][mem_col_addr] = data_in;
                        wr_en_signal = 2'b11;  // Write enable for memory bank
                    end
                    2'b01: begin // Kernel register
                        kernel_data_in = data_in;
                    end
                    2'b10: begin // Bias register
                        bias_data_in = data_in;
                    end
                endcase
            end
            default: begin
                // For other operations, default values hold
            end
        endcase
    end

    // Instantiate the memory_bank
    memory_bank mem_bank (
        .clk(clk),
        .rst_n(rst_n),
        .raddr(saddr),           // Connect to saddr for memory bank
        .waddr(mem_or_reg_addr), // Write to extracted address
        .wr_en(wr_en_signal),
        .row_addr(mem_row_addr), // Extracted row address
        .col_addr(mem_col_addr), // Extracted col address
        .data_in(mem_data_in),
        .data_out(data_from_mem)
    );
    // Instantiate the KernelRegister
    KernelRegister kernel_reg (
        .clk(clk),
        .rst_n(rst_n),
        .raddr(sub_addr),        // Connect to sub_addr for kernel register
        .waddr(mem_or_reg_addr), // Write to extracted address
        .wr_en(opcode == 3'b101 && mem_type_from_wb_addr == 2'b01), // Write enable only when opcode is Load and target is kernel register
        .row_addr(mem_row_addr), // Extracted row address
        .col_addr(mem_col_addr), // Extracted col address
        .data_in(kernel_data_in),
        .data_out(kernel_data)
    );
    // Instantiate the Bias_Register
    Bias_Register bias_reg (
        .clk(clk),
        .rst_n(rst_n),
        .raddr(sub_addr),        // Connect to sub_addr for bias register
        .waddr(mem_or_reg_addr), // Write to extracted address
        .wr_en(opcode == 3'b101 && mem_type_from_wb_addr == 2'b10), // Write enable only when opcode is Load and target is bias register
        .data_in(bias_data_in),
        .data_out(bias_data_from_reg)
    );
endmodule
```

### ChatGPT:
I understand your request. I will modify the `MemoryWrapper` module to ensure that for Convolution (Conv), Activation (Act), Pooling, and Bias Addition (Bias) operations, the `waddr` of the memory bank is `daddr`. For Load operations, the `waddr` will be `mem_or_reg_addr`. Here is the updated code:

```systemverilog
module MemoryWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [31:0] wb_addr,
    input logic [4:0] saddr,
    input logic [4:0] daddr, // Added daddr input
    input logic [4:0] sub_addr,
    input logic done,
    input logic [31:0] data_in, 
    input logic [31:0][31:0][31:0] padded_output, post_act_data, pooled_output_data, data_with_bias,
    output logic [31:0][31:0][31:0] data_from_mem,
    output logic [6:0][6:0][31:0] kernel_data,
    output logic [31:0] bias_data_from_reg
);

    // ... [rest of the declarations]

    // Logic for setting addresses, controlling writes, and feeding data
    always_comb begin
        // ... [rest of the logic]

        case (opcode)
            3'b001: begin // Conv
                if (done) begin
                    mem_data_in = padded_output;
                    wr_en_signal = 2'b10;       // Write enable for memory bank
                end
            end
            // ... [rest of the cases]
        endcase
    end

    // Instantiate the memory_bank
    memory_bank mem_bank (
        .clk(clk),
        .rst_n(rst_n),
        .raddr(saddr),           
        .waddr((opcode >= 3'b001 && opcode <= 3'b100) ? daddr : mem_or_reg_addr), // Updated waddr
        .wr_en(wr_en_signal),
        .row_addr(mem_row_addr), 
        .col_addr(mem_col_addr), 
        .data_in(mem_data_in),
        .data_out(data_from_mem)
    );

    // ... [rest of the module]

endmodule
```

In this updated code, the `waddr` for the memory bank (`mem_bank`) is determined based on the `opcode`. For Conv, Act, Pooling, and Bias operations (i.e., when `opcode` is between `3'b001` and `3'b100`), `waddr` is set to `daddr`. For other operations, `waddr` is set to `mem_or_reg_addr` as per the original logic.
