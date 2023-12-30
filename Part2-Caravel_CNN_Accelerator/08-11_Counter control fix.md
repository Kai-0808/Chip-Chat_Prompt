## 08-11_Counter control fix
### User:
Using `fsm_state_t` as input type is illegal. Please revise it.

Recall:
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

    logic [4:0] row_counter_next, col_counter_next;

    always_comb begin
        // Default assignments
        row_counter_next = row_counter;
        col_counter_next = col_counter;

        // Counter reset when transitioning back to IDLE
        if (next_state == IDLE) begin
            row_counter_next = 5'b0;
            col_counter_next = 5'b0;
        end
        // Counter logic for CONV_START state
        else if (current_state == CONV_START) begin
            if (stride) begin // stride is 2
                col_counter_next = col_counter + 2;
                if (col_counter_next > map_size) begin
                    col_counter_next = 0;
                    row_counter_next = row_counter + 2;
                end
            end else begin // stride is 1
                col_counter_next = col_counter + 1;
                if (col_counter_next == map_size + 1) begin
                    col_counter_next = 0;
                    row_counter_next = row_counter + 1;
                end
            end
        end
        // Counter logic for LOAD_START and STORE_START states
        else if (current_state == LOAD_START || current_state == STORE_START) begin
            col_counter_next = col_counter + 1;
            if (col_counter_next == map_size + 1) begin
                col_counter_next = 0;
                row_counter_next = row_counter + 1;
            end
        end
        // Add logic for other states as needed...
    end

    // Sequential block for counters
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            row_counter <= 5'b0;
            col_counter <= 5'b0;
        end else begin
            row_counter <= row_counter_next;
            col_counter <= col_counter_next;
        end
    end

endmodule
```

### ChatGPT:
I understand the issue. The `fsm_state_t` typedef should be declared before its usage in the module's port list. Additionally, using user-defined types (enums) as input ports is not recommended. Instead, we should use `logic` vectors and internally cast or compare them to the enum values.

Here's a revised version of the `CounterUnit`:

```systemverilog
module CounterUnit (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] next_state,
    input logic [4:0] map_size,
    input logic stride,
    output logic [4:0] row_counter,
    output logic [4:0] col_counter
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

    fsm_state_t current_fsm_state, next_fsm_state;
    logic [4:0] row_counter_next, col_counter_next;

    // Assign the logic vectors to the enum type for internal use
    assign current_fsm_state = current_state;
    assign next_fsm_state = next_state;

    always_comb begin
        // Default assignments
        row_counter_next = row_counter;
        col_counter_next = col_counter;

        // Counter reset when transitioning back to IDLE
        if (next_fsm_state == IDLE) begin
            row_counter_next = 5'b0;
            col_counter_next = 5'b0;
        end
        // Counter logic for CONV_START state
        else if (current_fsm_state == CONV_START) begin
            if (stride) begin // stride is 2
                col_counter_next = col_counter + 2;
                if (col_counter_next > map_size) begin
                    col_counter_next = 0;
                    row_counter_next = row_counter + 2;
                end
            end else begin // stride is 1
                col_counter_next = col_counter + 1;
                if (col_counter_next == map_size + 1) begin
                    col_counter_next = 0;
                    row_counter_next = row_counter + 1;
                end
            end
        end
        // Counter logic for LOAD_START and STORE_START states
        else if (current_fsm_state == LOAD_START || current_fsm_state == STORE_START) begin
            col_counter_next = col_counter + 1;
            if (col_counter_next == map_size + 1) begin
                col_counter_next = 0;
                row_counter_next = row_counter + 1;
            end
        end
        // Add logic for other states as needed...
    end

    // Sequential block for counters
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            row_counter <= 5'b0;
            col_counter <= 5'b0;
        end else begin
            row_counter <= row_counter_next;
            col_counter <= col_counter_next;
        end
    end

endmodule
```

This version uses `logic [2:0]` for the inputs `current_state` and `next_state`, and then internally assigns them to the `fsm_state_t` type for more readable comparisons and logic operations.

### User:
    assign current_fsm_state = current_state;
                                           |
ncvlog: *W,ENUMERR (CounterUnit.sv,27|43): This assignment is a violation of SystemVerilog strong typing rules for enumeration datatypes.

### ChatGPT:
I apologize for the oversight. You're right; direct assignments between enum types and logic vectors can raise strong typing issues in some SystemVerilog compilers.

Instead, we can avoid the direct assignment by directly using the logic vectors in comparisons with the enumerated values. Here's the modified version:

```systemverilog
module CounterUnit (
    input logic clk,
    input logic rst_n,
    input logic [2:0] current_state,
    input logic [2:0] next_state,
    input logic [4:0] map_size,
    input logic stride,
    output logic [4:0] row_counter,
    output logic [4:0] col_counter
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

    logic [4:0] row_counter_next, col_counter_next;

    always_comb begin
        // Default assignments
        row_counter_next = row_counter;
        col_counter_next = col_counter;

        // Counter reset when transitioning back to IDLE
        if (next_state == IDLE) begin
            row_counter_next = 5'b0;
            col_counter_next = 5'b0;
        end
        // Counter logic for CONV_START state
        else if (current_state == CONV_START) begin
            if (stride) begin // stride is 2
                col_counter_next = col_counter + 2;
                if (col_counter_next > map_size) begin
                    col_counter_next = 0;
                    row_counter_next = row_counter + 2;
                end
            end else begin // stride is 1
                col_counter_next = col_counter + 1;
                if (col_counter_next == map_size + 1) begin
                    col_counter_next = 0;
                    row_counter_next = row_counter + 1;
                end
            end
        end
        // Counter logic for LOAD_START and STORE_START states
        else if (current_state == LOAD_START || current_state == STORE_START) begin
            col_counter_next = col_counter + 1;
            if (col_counter_next == map_size + 1) begin
                col_counter_next = 0;
                row_counter_next = row_counter + 1;
            end
        end
        // Add logic for other states as needed...
    end

    // Sequential block for counters
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            row_counter <= 5'b0;
            col_counter <= 5'b0;
        end else begin
            row_counter <= row_counter_next;
            col_counter <= col_counter_next;
        end
    end

endmodule
```

In this version, we directly use `current_state` and `next_state` logic vectors for comparisons, thus avoiding any strong typing issues.
