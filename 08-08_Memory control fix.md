## 08-08_Memory control fix
### User:
For the memory module, the inputs `saddr` and `sub_addr` is missing. Also, `wb_addr` is 32-bit and `data_in` is a single 32-bit signal, too.
The assignment `mem_data_in = data_in;` inside `case (mem_type_from_wb_addr)` should determine a correct position based on `mem_row/col_addr`.

Recall:
```systemverilog
module MemoryWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [4:0] map_size,
    input logic [4:0] wb_addr,
    input logic done,
    input logic [31:0][31:0][31:0] data_in, padded_output, post_act_data, pooled_output_data, data_with_bias,
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
                    mem_data_in = data_in;
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
Here's the updated `MemoryWrapper` module with the necessary changes:

1. Added the `saddr` and `sub_addr` inputs.
2. Adjusted the assignment of `mem_data_in` based on `mem_row/col_addr`.

```systemverilog
module MemoryWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [4:0] map_size,
    input logic [31:0] wb_addr,
    input logic [4:0] saddr,
    input logic [4:0] sub_addr,
    input logic done,
    input logic [31:0] data_in, 
    output logic [31:0][31:0][31:0] data_from_mem, post_act_data, padded_output, pooled_output_data, data_with_bias,
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
                    wr_en_signal = 2'b10; // Write enable for memory bank
                end
            end
            3'b010: begin // Activation
                if (done) begin
                    mem_data_in = post_act_data; 
                    wr_en_signal = 2'b10; // Write enable for memory bank
                end
            end
            3'b011: begin // Pooling
                if (done) begin
                    mem_data_in = pooled_output_data;
                    wr_en_signal = 2'b10; // Write enable for memory bank
                end
            end
            3'b100: begin // Bias
                if (done) begin
                    mem_data_in = data_with_bias;
                    wr_en_signal = 2'b10; // Write enable for memory bank
                end
            end
            3'b101: begin // Load
                case (mem_type_from_wb_addr)
                    2'b00: begin // Memory bank
                        mem_data_in[mem_row_addr][mem_col_addr] = data_in;
                        wr_en_signal = 2'b11; // Write enable for memory bank
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
        .raddr(saddr),
        .waddr(mem_or_reg_addr),
        .wr_en(wr_en_signal),
        .row_addr(mem_row_addr),
        .col_addr(mem_col_addr),
        .data_in(mem_data_in),
        .data_out(data_from_mem)
    );

    // Instantiate the KernelRegister
    KernelRegister kernel_reg (
        .clk(clk),
        .rst_n(rst_n),
        .raddr(sub_addr),
        .waddr(mem_or_reg_addr),
        .wr_en(opcode == 3'b101 && mem_type_from_wb_addr == 2'b01),
        .row_addr(mem_row_addr),
        .col_addr(mem_col_addr),
        .data_in(kernel_data_in),
        .data_out(kernel_data)
    );

    // Instantiate the Bias_Register
    Bias_Register bias_reg (
        .clk(clk),
        .rst_n(rst_n),
        .raddr(sub_addr),
        .waddr(mem_or_reg_addr),
        .wr_en(opcode == 3'b101 && mem_type_from_wb_addr == 2'b10),
        .data_in(bias_data_in),
        .data_out(bias_data_from_reg)
    );
endmodule
```

This updated module should now correctly handle the inputs and perform memory operations based on the provided opcode and address signals.
