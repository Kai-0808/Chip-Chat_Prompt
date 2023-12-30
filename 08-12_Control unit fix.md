## 08-12_Control unit fix
### User:
Please revise the instantiation of MemoryWrapper:
module ControlUnit (
    input logic clk,
    input logic rst_n,
    input logic [2:0] opcode,
    input logic [4:0] saddr,
    input logic [4:0] daddr,
    input logic [4:0] map_size,
    input logic [4:0] sub_addr,
    input logic [2:0] sub_size,
    input logic stride,
    input logic [1:0] padding,
    inout logic [31:0] wb_addr,
    input logic [31:0] data_in,
    input logic [1:0] mem_type,
    output logic [31:0] data_out,
    output logic done,
    output logic out_valid
);

    // Internal signals
    logic [2:0] current_state, next_state;
    logic [4:0] row_counter, col_counter;
    logic [31:0][31:0][31:0] data_from_mem, post_act_data, padded_output, pooled_output_data, data_with_bias;
    logic [6:0][6:0][31:0] kernel_data;
    logic [31:0] bias_data_from_reg;
    logic done_conv;

    // Instantiate the StateMachine
    StateMachine sm (
        .clk(clk),
        .rst_n(rst_n),
        .opcode(opcode),
        .mem_type(mem_type),
        .sub_size(sub_size),
        .padding(padding),
        .map_size(map_size),
        .row_counter(row_counter),
        .col_counter(col_counter),
        .done_conv(done_conv),
        .current_state(current_state),
        .next_state(next_state)
    );

    // Instantiate the CounterUnit
    CounterUnit cu (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .next_state(next_state),
        .map_size(map_size),
        .stride(stride),
        .row_counter(row_counter),
        .col_counter(col_counter)
    );

    // Instantiate the ActivationWrapper
    ActivationWrapper aw (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .opcode(opcode),
        .data_from_mem(data_from_mem),
        .post_act_data(post_act_data)
    );

    // Instantiate the BiasAdditionWrapper
    BiasAdditionWrapper baw (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .opcode(opcode),
        .map_size(map_size),
        .data_from_mem(data_from_mem),
        .bias_data_from_reg(bias_data_from_reg),
        .data_with_bias(data_with_bias)
    );

    // Instantiate the ConvModule
    ConvModule cm (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .opcode(opcode),
        .data_from_mem(data_from_mem),
        .kernel_data(kernel_data),
        .stride(stride),
        .sub_size(sub_size),
        .padding(padding),
        .map_size(map_size),
        .row_counter(row_counter),
        .col_counter(col_counter),
        .padded_output(padded_output),
        .done_conv(done_conv)
    );

    // Instantiate the MemoryWrapper
    MemoryWrapper mw (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .opcode(opcode),
        .map_size(map_size),
        .wb_addr(wb_addr),
        .done(done),
        .data_in(data_in),
        .padded_output(padded_output),
        .post_act_data(post_act_data),
        .pooled_output_data(pooled_output_data),
        .data_with_bias(data_with_bias),
        .data_from_mem(data_from_mem),
        .kernel_data(kernel_data),
        .bias_data_from_reg(bias_data_from_reg)
    );

    // Instantiate the PoolingWrapper
    PoolingWrapper pw (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .opcode(opcode),
        .map_size(map_size),
        .stride(stride),
        .sub_size(sub_size),
        .data_from_mem(data_from_mem),
        .pooled_output_data(pooled_output_data)
    );

    // Output logic
    always_comb begin
        // Signal for operation completion
        done = (next_state == IDLE) && (current_state != IDLE && current_state != FAULT);
        // Signal to indicate valid data on data_out during a store operation
        out_valid = (next_state == STORE_START);
        // Default assignment for data_out
        data_out = 32'b0;
        // Assign data_out based on state and operation
        if (out_valid) begin
            case (mem_type)
                2'b00: data_out = data_from_mem[row_counter][col_counter]; // Fetching from memory bank
                2'b01: data_out = kernel
endmodule
New code of MemoryWrapper:
module MemoryWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [31:0] wb_addr,
    input logic [4:0] saddr,
    input logic [4:0] daddr,
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
        .waddr((opcode >= 3'b001 && opcode <= 3'b100) ? daddr : mem_or_reg_addr), // Updated waddr
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

### ChatGPT:
The given changes are implemented in the MemoryWrapper instantiation. For the memory bank's write address (`waddr`), we now use `daddr` for convolution, activation, pooling, and bias operations. For load operations, we use the extracted address `mem_or_reg_addr`.

Here's the revised instantiation of `MemoryWrapper` inside the `ControlUnit`:

```systemverilog
module ControlUnit (
    input logic clk,
    input logic rst_n,
    input logic [2:0] opcode,
    input logic [4:0] saddr,
    input logic [4:0] daddr,
    input logic [4:0] map_size,
    input logic [4:0] sub_addr,
    input logic [2:0] sub_size,
    input logic stride,
    input logic [1:0] padding,
    inout logic [31:0] wb_addr,
    input logic [31:0] data_in,
    input logic [1:0] mem_type,
    output logic [31:0] data_out,
    output logic done,
    output logic out_valid
);

    // [rest of the module's internals remain unchanged]

    // Instantiate the MemoryWrapper
    MemoryWrapper mw (
        .clk(clk),
        .rst_n(rst_n),
        .current_state(current_state),
        .opcode(opcode),
        .wb_addr(wb_addr),
        .saddr(saddr),
        .daddr(daddr),
        .sub_addr(sub_addr),
        .done(done),
        .data_in(data_in),
        .padded_output(padded_output),
        .post_act_data(post_act_data),
        .pooled_output_data(pooled_output_data),
        .data_with_bias(data_with_bias),
        .data_from_mem(data_from_mem),
        .kernel_data(kernel_data),
        .bias_data_from_reg(bias_data_from_reg)
    );

    // [rest of the module's internals remain unchanged]

endmodule
```

This instantiation ensures that the correct addresses are fed to the `MemoryWrapper` for read and write operations based on the operation type.
