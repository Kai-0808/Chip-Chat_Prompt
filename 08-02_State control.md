## 08-02_State control
### User:
For the state and intruction fail part, wrap it into a module that can be instantiated by ControlUnit. I need this module to output current state and next state. 

Recall:
```systemverilog
typedef enum logic [2:0] {
    IDLE        = 3'b000,
    CONV_START  = 3'b001,
    ACT_START   = 3'b010,
    POOL_START  = 3'b011,
    BIAS_START  = 3'b100,
    LOAD_START  = 3'b101,
    STORE_START = 3'b110,
    FAULT       = 3'b111
} fsm_state_t;

fsm_state_t current_state, next_state;

// Check for invalid instruction
always_comb begin
    instr_fault = 0;
    case (opcode)
        3'b001: begin
            if (sub_size != 3'd1 && sub_size != 3'd3 && sub_size != 3'd5 && sub_size != 3'd7) instr_fault = 1;
            if (padding != 2'b00 && padding != 2'b01) instr_fault = 1;
        end
        3'b011: if (sub_size != 3'd2 && sub_size != 3'd3) instr_fault = 1;
        default: if (opcode > 3'b110) instr_fault = 1;
    endcase
    if (mem_type > 2'b10) instr_fault = 1;
end

always_comb begin
    // Default next state assignment
    next_state = current_state;
    // FSM transitions based on current state and opcode
    case (current_state)
        IDLE: begin
            if (instr_fault) next_state = FAULT;
            else case (opcode)
                3'b001: next_state = CONV_START;
                3'b010: next_state = ACT_START;
                3'b011: next_state = POOL_START;
                3'b100: next_state = BIAS_START;
                3'b101: next_state = LOAD_START;
                3'b110: next_state = STORE_START;
                default: next_state = FAULT;
            endcase
        end
        CONV_START: if (done_conv) next_state = IDLE; // Remain in CONV_START until convolution is done
        ACT_START: next_state = IDLE;  // Transition back to IDLE after initiating activation
        POOL_START: next_state = IDLE; // Transition back to IDLE after initiating pooling
        BIAS_START: next_state = IDLE; // Transition back to IDLE after initiating bias addition
        LOAD_START: if (row_counter == map_size && col_counter == map_size) next_state = IDLE; // Remain in LOAD_START until counters reach end
        STORE_START: if (row_counter == map_size && col_counter == map_size) next_state = IDLE; // Remain in STORE_START until counters reach end
        FAULT: next_state = IDLE; // Transition back to IDLE after a fault
    endcase
end
```

### ChatGPT:
Here's the FSM module for state management and instruction fault checking:

```systemverilog
module StateMachine (
    input logic clk,
    input logic rst_n,
    input logic [2:0] opcode,
    input logic [2:0] mem_type,
    input logic [2:0] sub_size,
    input logic [1:0] padding,
    input logic [4:0] row_counter,
    input logic [4:0] col_counter,
    input logic done_conv,
    output logic [2:0] current_state,
    output logic [2:0] next_state,
    output logic instr_fault
);

    typedef enum logic [2:0] {
        IDLE        = 3'b000,
        CONV_START  = 3'b001,
        ACT_START   = 3'b010,
        POOL_START  = 3'b011,
        BIAS_START  = 3'b100,
        LOAD_START  = 3'b101,
        STORE_START = 3'b110,
        FAULT       = 3'b111
    } fsm_state_t;

    fsm_state_t current_state_reg, next_state_reg;

    // Check for invalid instruction
    always_comb begin
        instr_fault = 0;
        case (opcode)
            3'b001: begin
                if (sub_size != 3'd1 && sub_size != 3'd3 && sub_size != 3'd5 && sub_size != 3'd7) instr_fault = 1;
                if (padding != 2'b00 && padding != 2'b01) instr_fault = 1;
            end
            3'b011: if (sub_size != 3'd2 && sub_size != 3'd3) instr_fault = 1;
            default: if (opcode > 3'b110) instr_fault = 1;
        endcase
        if (mem_type > 2'b10) instr_fault = 1;
    end

    always_comb begin
        // Default next state assignment
        next_state_reg = current_state_reg;
        // FSM transitions based on current state and opcode
        case (current_state_reg)
            IDLE: begin
                if (instr_fault) next_state_reg = FAULT;
                else case (opcode)
                    3'b001: next_state_reg = CONV_START;
                    3'b010: next_state_reg = ACT_START;
                    3'b011: next_state_reg = POOL_START;
                    3'b100: next_state_reg = BIAS_START;
                    3'b101: next_state_reg = LOAD_START;
                    3'b110: next_state_reg = STORE_START;
                    default: next_state_reg = FAULT;
                endcase
            end
            CONV_START: if (done_conv) next_state_reg = IDLE;
            ACT_START: next_state_reg = IDLE;
            POOL_START: next_state_reg = IDLE;
            BIAS_START: next_state_reg = IDLE;
            LOAD_START: if (row_counter == map_size && col_counter == map_size) next_state_reg = IDLE;
            STORE_START: if (row_counter == map_size && col_counter == map_size) next_state_reg = IDLE;
            FAULT: next_state_reg = IDLE;
        endcase
    end

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            current_state_reg <= IDLE;
        else
            current_state_reg <= next_state_reg;
    end

    assign current_state = current_state_reg;
    assign next_state = next_state_reg;

endmodule
```

This module captures the essence of state management and fault detection logic. It can be instantiated in the `ControlUnit` for proper operation sequencing and fault handling.

### User:
Let the `instr_fault` be internal signal.

### ChatGPT:
Certainly. Here's the modified module with `instr_fault` being internal:

```systemverilog
module StateMachine (
    input logic clk,
    input logic rst_n,
    input logic [2:0] opcode,
    input logic [2:0] mem_type,
    input logic [2:0] sub_size,
    input logic [1:0] padding,
    input logic [4:0] row_counter,
    input logic [4:0] col_counter,
    input logic done_conv,
    output logic [2:0] current_state,
    output logic [2:0] next_state
);

    typedef enum logic [2:0] {
        IDLE        = 3'b000,
        CONV_START  = 3'b001,
        ACT_START   = 3'b010,
        POOL_START  = 3'b011,
        BIAS_START  = 3'b100,
        LOAD_START  = 3'b101,
        STORE_START = 3'b110,
        FAULT       = 3'b111
    } fsm_state_t;

    fsm_state_t current_state_reg, next_state_reg;
    logic instr_fault;  // Internal signal for instruction fault

    // Check for invalid instruction
    always_comb begin
        instr_fault = 0;
        case (opcode)
            3'b001: begin
                if (sub_size != 3'd1 && sub_size != 3'd3 && sub_size != 3'd5 && sub_size != 3'd7) instr_fault = 1;
                if (padding != 2'b00 && padding != 2'b01) instr_fault = 1;
            end
            3'b011: if (sub_size != 3'd2 && sub_size != 3'd3) instr_fault = 1;
            default: if (opcode > 3'b110) instr_fault = 1;
        endcase
        if (mem_type > 2'b10) instr_fault = 1;
    end

    always_comb begin
        // Default next state assignment
        next_state_reg = current_state_reg;
        // FSM transitions based on current state and opcode
        case (current_state_reg)
            IDLE: begin
                if (instr_fault) next_state_reg = FAULT;
                else case (opcode)
                    3'b001: next_state_reg = CONV_START;
                    3'b010: next_state_reg = ACT_START;
                    3'b011: next_state_reg = POOL_START;
                    3'b100: next_state_reg = BIAS_START;
                    3'b101: next_state_reg = LOAD_START;
                    3'b110: next_state_reg = STORE_START;
                    default: next_state_reg = FAULT;
                endcase
            end
            CONV_START: if (done_conv) next_state_reg = IDLE;
            ACT_START: next_state_reg = IDLE;
            POOL_START: next_state_reg = IDLE;
            BIAS_START: next_state_reg = IDLE;
            LOAD_START: if (row_counter == map_size && col_counter == map_size) next_state_reg = IDLE;
            STORE_START: if (row_counter == map_size && col_counter == map_size) next_state_reg = IDLE;
            FAULT: next_state_reg = IDLE;
        endcase
    end

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            current_state_reg <= IDLE;
        else
            current_state_reg <= next_state_reg;
    end

    assign current_state = current_state_reg;
    assign next_state = next_state_reg;

endmodule
```

I've removed `instr_fault` from the module's output ports and kept it internal.

### User:
Input map_size

### ChatGPT:
Sure, I'll incorporate `map_size` as an input to the module. 

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

    typedef enum logic [2:0] {
        IDLE        = 3'b000,
        CONV_START  = 3'b001,
        ACT_START   = 3'b010,
        POOL_START  = 3'b011,
        BIAS_START  = 3'b100,
        LOAD_START  = 3'b101,
        STORE_START = 3'b110,
        FAULT       = 3'b111
    } fsm_state_t;

    fsm_state_t current_state_reg, next_state_reg;
    logic instr_fault;  // Internal signal for instruction fault

    // Check for invalid instruction
    always_comb begin
        instr_fault = 0;
        case (opcode)
            3'b001: begin
                if (sub_size != 3'd1 && sub_size != 3'd3 && sub_size != 3'd5 && sub_size != 3'd7) instr_fault = 1;
                if (padding != 2'b00 && padding != 2'b01) instr_fault = 1;
            end
            3'b011: if (sub_size != 3'd2 && sub_size != 3'd3) instr_fault = 1;
            default: if (opcode > 3'b110) instr_fault = 1;
        endcase
        if (mem_type > 2'b10) instr_fault = 1;
    end

    always_comb begin
        // Default next state assignment
        next_state_reg = current_state_reg;
        // FSM transitions based on current state and opcode
        case (current_state_reg)
            IDLE: begin
                if (instr_fault) next_state_reg = FAULT;
                else case (opcode)
                    3'b001: next_state_reg = CONV_START;
                    3'b010: next_state_reg = ACT_START;
                    3'b011: next_state_reg = POOL_START;
                    3'b100: next_state_reg = BIAS_START;
                    3'b101: next_state_reg = LOAD_START;
                    3'b110: next_state_reg = STORE_START;
                    default: next_state_reg = FAULT;
                endcase
            end
            CONV_START: if (done_conv) next_state_reg = IDLE;
            ACT_START: next_state_reg = IDLE;
            POOL_START: next_state_reg = IDLE;
            BIAS_START: next_state_reg = IDLE;
            LOAD_START: if (row_counter == map_size && col_counter == map_size) next_state_reg = IDLE;
            STORE_START: if (row_counter == map_size && col_counter == map_size) next_state_reg = IDLE;
            FAULT: next_state_reg = IDLE;
        endcase
    end

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            current_state_reg <= IDLE;
        else
            current_state_reg <= next_state_reg;
    end

    assign current_state = current_state_reg;
    assign next_state = next_state_reg;

endmodule
```

I've added `map_size` as an input to the module.
