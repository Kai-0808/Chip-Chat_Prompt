## 08-06_Counter control
### User:
For the counter part, wrap it into a module that can be instantiated by ControlUnit. Consider `current_state` and `next_state` as inputs instead. Make sure the necessary input and output ports being inculded in the module.
Recall:
```systemverilog
logic [4:0] row_counter, col_counter;
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
```

### ChatGPT:
Let's wrap the counter logic into a module. The module will have the following ports:

Inputs:
- Clock and reset signals (`clk`, `rst_n`)
- Current and next states (`current_state`, `next_state`)
- Map size (`map_size`)
- Stride (`stride`)

Outputs:
- Row and column counters (`row_counter`, `col_counter`)

Here's how the module will look:

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

This module encapsulates the counter logic and can be instantiated in the ControlUnit. The module takes in the necessary inputs to determine the counter values and outputs the row and column counters.
