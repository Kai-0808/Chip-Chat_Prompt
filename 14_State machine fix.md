## 14_State machine fix
### User:
module StateMachine (
    input logic clk,
    input logic rst_n,
    input logic [2:0] opcode,
    input logic [1:0] mem_type,
    input logic [2:0] sub_size,
    input logic [1:0] padding,
    input logic [4:0] map_size,
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

In the above is my StateMachine design. I want you to focus on the instr fault part. This module can't actually report instr fault because the input signal rises and falls on negedge of clk, and thus makes the FF can't fetch it on posedge of clk. Please fix this.

### ChatGPT:
To handle the instruction fault signal, which changes on the negative edge of the clock, you should sample the `instr_fault` signal on the negative edge and store it in a flip-flop. Then, you can use the stored value in the FSM logic, which operates on the positive edge of the clock.

Here's one way to do it:

1. Add a flip-flop to sample the `instr_fault` on the negative edge.
2. Use the sampled `instr_fault` in your FSM logic.

Here's the modification to your `StateMachine` module:

```systemverilog
module StateMachine (
    // ... [rest of the ports as before]
);

    // ... [rest of the typedef and variable declarations as before]

    logic instr_fault_sampled; // Sampled version of instr_fault

    // Sample instr_fault on the negative edge
    always_ff @(negedge clk or negedge rst_n) begin
        if (!rst_n)
            instr_fault_sampled <= 0;
        else
            instr_fault_sampled <= instr_fault;
    end

    always_comb begin
        // ... [rest of the instr_fault logic as before]
    end

    always_comb begin
        // Default next state assignment
        next_state_reg = current_state_reg;
        // FSM transitions based on current state and opcode
        case (current_state_reg)
            IDLE: begin
                if (instr_fault_sampled) next_state_reg = FAULT; // Use instr_fault_sampled here
                else case (opcode)
                    // ... [rest of the FSM logic as before]
                endcase
            end
            // ... [rest of the FSM states as before]
        endcase
    end

    // ... [rest of the always_ff block and assignments as before]

endmodule
```

With these changes, the `instr_fault` signal is sampled on the negative edge of the clock and stored in `instr_fault_sampled`. This allows the FSM logic, which operates on the positive edge, to use the sampled value.

### User:
I think we should do the sample on posedge of clk in order to make it reach the next posedge of clk?

### ChatGPT:
You're right; if the `instr_fault` signal changes on the negative edge of the clock, and you want to make it available for the positive edge FSM logic in the next cycle, you should sample it on the positive edge of the clock. My apologies for the oversight.

Here's the corrected modification:

1. Sample the `instr_fault` signal on the positive edge of the clock.
2. Use the sampled `instr_fault` in your FSM logic.

```systemverilog
module StateMachine (
    // ... [rest of the ports as before]
);

    // ... [rest of the typedef and variable declarations as before]

    logic instr_fault_sampled; // Sampled version of instr_fault

    // Sample instr_fault on the positive edge
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            instr_fault_sampled <= 0;
        else
            instr_fault_sampled <= instr_fault;
    end

    always_comb begin
        // ... [rest of the instr_fault logic as before]
    end

    always_comb begin
        // Default next state assignment
        next_state_reg = current_state_reg;
        // FSM transitions based on current state and opcode
        case (current_state_reg)
            IDLE: begin
                if (instr_fault_sampled) next_state_reg = FAULT; // Use instr_fault_sampled here
                else case (opcode)
                    // ... [rest of the FSM logic as before]
                endcase
            end
            // ... [rest of the FSM states as before]
        endcase
    end

    // ... [rest of the always_ff block and assignments as before]

endmodule
```

By sampling `instr_fault` on the positive edge, it ensures that the FSM logic, which also operates on the positive edge, will see the updated value in the next cycle.
