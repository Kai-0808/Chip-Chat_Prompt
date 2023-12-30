## 08-07_Control unit integration
### User:
Integrate the following partition into the ControlUnit.

### ControlUnit:
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
    output logic out_valid,
    output logic instr_fault
);
```
The interface of the submodules are shown below.

**ActivationWrapper:**
```systemverilog
module ActivationWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [31:0][31:0][31:0] data_from_mem,
    output logic [31:0][31:0][31:0] post_act_data
);
```

**BiasAdditionWrapper:**
```systemverilog
module BiasAdditionWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [4:0] map_size,
    input logic [31:0][31:0][31:0] data_from_mem,
    input logic [31:0] bias_data_from_reg,
    output logic [31:0][31:0][31:0] data_with_bias
);
```

**ConvModule:**
```systemverilog
module ConvModule (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [31:0][31:0][31:0] data_from_mem,
    input logic [6:0][6:0][31:0] kernel_data,
    input logic stride,
    input logic [2:0] sub_size,
    input logic [1:0] padding,
    input logic [4:0] map_size,
    input logic [4:0] row_counter,
    input logic [4:0] col_counter,
    output logic [31:0][31:0][31:0] padded_output,
    output logic done_conv
);
```

**CounterUnit:**
```systemverilog
module CounterUnit (
    input logic clk,
    input logic rst_n,
    input fsm_state_t current_state,
    input fsm_state_t next_state,
    input logic [4:0] map_size,
    input logic stride,
    output logic [4:0] row_counter,
    output logic [4:0] col_counter
);
```

**MemoryWrapper:**
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
```

**PoolingWrapper:**
```systemverilog
module PoolingWrapper (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] opcode,
    input logic [4:0] map_size,
    input logic stride,
    input logic [2:0] sub_size,
    input logic [31:0][31:0][31:0] data_from_mem,
    output logic [31:0][31:0][31:0] pooled_output_data
);
```

**StateMachine:**
```systemverilog
module StateMachine (
    input logic clk,
    input logic rst_n,
    input logic [2:0] opcode,
    input logic [2:0] mem_type,
    input logic [2:0] sub_size,
    input logic [1:0] padding,
    input logic [4:0] map_size,          // Added map_size as an input
    input logic [4:0] row_counter,
    input logic [4:0] col_counter,
    input logic done_conv,
    output logic [2:0] current_state,
    output logic [2:0] next_state
);
```

Also, integrate the following output logic into the ControlUnit.
```systemverilog
always_comb begin
    // Signal for operation completion
    done = (next_state == IDLE) && (current_state != IDLE && current_state != FAULT);
    // Signal to indicate valid data on data_out during a store operation
    out_valid = (next_state == STORE_START);
    // Default assignment for data_out
    data_out = 32'b0;
    // Assign data_out based on state and operation
    if (out_valid) begin
        case (mem_type_from_wb_addr)
            2'b00: data_out = data_from_mem[row_counter][col_counter]; // Fetching from memory bank
            2'b01: data_out = kernel_data[row_counter][col_counter];   // Fetching from kernel register
            2'b10: data_out = bias_data_from_reg;                      // Fetching from bias register
            default: data_out = 32'b0;                                 // Default value
        endcase
    end
end
```

### ChatGPT:
Given the provided interfaces for the sub-modules, let's integrate them into the `ControlUnit`. 

I'll instantiate each of the modules within the `ControlUnit` and wire up the necessary connections based on the provided interfaces.

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
    output logic out_valid,
    output logic instr_fault
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
                2'b01: data_out = kernel_data[row_counter][col_counter];   // Fetching from kernel register
                2'b10: data_out = bias_data_from_reg;                      // Fetching from bias register
                default: data_out = 32'b0;                                 // Default value
            endcase
        end
    end

endmodule
```

This integrates all the provided modules into the `ControlUnit`. The signals between the modules are connected based on the interfaces provided.

### User:
Remove the output logic `instr_fault`.

### ChatGPT:
Alright, I'll remove the `instr_fault` output logic from the `ControlUnit`. 

Here's the updated `ControlUnit` without the `instr_fault` signal:

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
