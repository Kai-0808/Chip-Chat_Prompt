## 08-03_Pooling control
### User:
For the PoolingUnit part, wrap it into a module that can be instantiated by ControlUnit. Remove the state machine from the module, and consider `current_state` as an input instead. Make sure the necessary input and output ports being inculded in the module.
Recall:
```systemverilog
// Signals for PoolingUnit
logic [31:0][31:0][31:0] pooling_input_data, pooled_output_data;
logic encoded_window_size;

// Logic for feeding data and setting window size
always_comb begin
    if (current_state == IDLE && opcode == 3'b011) begin // When in IDLE and opcode is Pooling
        pooling_input_data = data_from_mem; // Assuming data_from_mem is connected to memory bank's output
        // Encoding the window size
        encoded_window_size = (sub_size == 3'd2) ? 1'b0 : 1'b1; // 1'b0 for 2x2, 1'b1 for 3x3
    end else begin
        pooling_input_data = 32'b0;
        encoded_window_size = 1'b0; // Default value
    end
end

// Instantiate the PoolingUnit
PoolingUnit pooling_unit (
    .clk(clk),
    .rst_n(rst_n),
    .Data(pooling_input_data),
    .Length(map_size),
    .Stride(stride),
    .WindowSize(encoded_window_size),
    .PooledData(pooled_output_data)
);

// FSM Transitions for Pooling (Partial)
always_comb begin
    // Default next state assignment
    next_state = current_state;
    // FSM transitions based on current state and opcode
    case (current_state)
        IDLE: begin
            if (instr_fault) next_state = FAULT;
            else case (opcode)
                3'b011: next_state = POOL_START;
                // ... [Rest of the IDLE state logic]
            endcase
        end
        POOL_START: next_state = IDLE; // Transition back to IDLE after pooling
        // ... [Rest of the FSM logic]
    endcase
end
```

### ChatGPT:
Let's wrap the `PoolingUnit` logic into a module:

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

    // Signals for PoolingUnit
    logic [31:0][31:0][31:0] pooling_input_data;
    logic encoded_window_size;

    // Logic for feeding data and setting window size
    always_comb begin
        if (current_state == 3'b000 && opcode == 3'b011) begin // When in IDLE and opcode is Pooling
            pooling_input_data = data_from_mem; // Using the provided data from memory
            // Encoding the window size
            encoded_window_size = (sub_size == 3'd2) ? 1'b0 : 1'b1; // 1'b0 for 2x2, 1'b1 for 3x3
        end else begin
            pooling_input_data = 32'b0;
            encoded_window_size = 1'b0; // Default value
        end
    end

    // Instantiate the PoolingUnit
    PoolingUnit pooling_unit (
        .clk(clk),
        .rst_n(rst_n),
        .Data(pooling_input_data),
        .Length(map_size),
        .Stride(stride),
        .WindowSize(encoded_window_size),
        .PooledData(pooled_output_data)
    );

endmodule
```

This `PoolingWrapper` module assumes that the `PoolingUnit` is already defined elsewhere. The FSM transitions specific to the pooling function have been removed from this module since it's expected that the ControlUnit will handle the state transitions.
