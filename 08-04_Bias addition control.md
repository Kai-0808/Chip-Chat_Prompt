## 08-04_Bias addition control
### User:
For the BiasAdditionUnit part, wrap it into a module that can be instantiated by ControlUnit. Remove the state machine from the module, and consider `current_state` as an input instead. Make sure the necessary input and output ports being inculded in the module.
Recall:
```systemverilog
// Signals for BiasAdditionUnit
logic [31:0][31:0][31:0] bias_input_data, data_with_bias;
logic [31:0] bias_value;

// Logic for feeding data and bias
always_comb begin
    if (current_state == IDLE && opcode == 3'b100) begin // When in IDLE and opcode is Bias
        bias_input_data = data_from_mem;   // Assuming data_from_mem is connected to memory bank's output
        bias_value = bias_data_from_reg;   // Assuming bias_data_from_reg is connected to bias register's output
    end else begin
        bias_input_data = 32'b0;
        bias_value = 32'b0; // Default value
    end
end

// Instantiate the BiasAdditionUnit
BiasAdditionUnit bias_unit (
    .clk(clk),
    .rst_n(rst_n),
    .Data(bias_input_data),
    .Length(map_size),
    .BiasValue(bias_value),
    .DataWithBias(data_with_bias)
);

// FSM Transitions for Bias Addition (Partial)
always_comb begin
    // Default next state assignment
    next_state = current_state;
    // FSM transitions based on current state and opcode
    case (current_state)
        IDLE: begin
            if (instr_fault) next_state = FAULT;
            else case (opcode)
                3'b100: next_state = BIAS_START;
                // ... [Rest of the IDLE state logic]
            endcase
        end
        BIAS_START: next_state = IDLE; // Transition back to IDLE after bias addition
        // ... [Rest of the FSM logic]
    endcase
end
```

### ChatGPT:
Let's wrap the `BiasAdditionUnit` logic into a module:

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

    // Signals for BiasAdditionUnit
    logic [31:0][31:0][31:0] bias_input_data;
    logic [31:0] bias_value;

    // Logic for feeding data and bias
    always_comb begin
        if (current_state == 3'b000 && opcode == 3'b100) begin // When in IDLE and opcode is Bias
            bias_input_data = data_from_mem;   // Using the provided data from memory
            bias_value = bias_data_from_reg;   // Using the provided bias data from register
        end else begin
            bias_input_data = 32'b0;
            bias_value = 32'b0; // Default value
        end
    end

    // Instantiate the BiasAdditionUnit
    BiasAdditionUnit bias_unit (
        .clk(clk),
        .rst_n(rst_n),
        .Data(bias_input_data),
        .Length(map_size),
        .BiasValue(bias_value),
        .DataWithBias(data_with_bias)
    );

endmodule
```

This `BiasAdditionWrapper` module assumes that the `BiasAdditionUnit` is already defined elsewhere. The FSM transitions specific to the bias addition function have been removed from this module since it's expected that the ControlUnit will handle the state transitions.
