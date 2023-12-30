## 13_Add power pins
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.
## To-Do:
Add power pin declarations below to the `Wishbone2Top` module.

### power pins:
```verilog
`ifdef USE_POWER_PINS
    inout vccd1,	// User area 1 1.8V supply
    inout vssd1,	// User area 1 digital ground
`endif
```

### Wishbone2Top:
```systemverilog
module Wishbone2Top(
    input wb_clk_i,
    input wb_rst_i,
    input wbs_stb_i,
    input wbs_cyc_i,
    input wbs_we_i,
    input [3:0] wbs_sel_i,
    input [31:0] wbs_dat_i,
    input [31:0] wbs_adr_i,
    output wbs_ack_o,
    output [31:0] wbs_dat_o
);

// Internal signals
logic data_valid;
logic [31:0] wb_adr_out, data;
logic [31:0] data_out;
logic out_valid, full;

// FSM states
typedef enum {IDLE, WRITE, READ} fsm_state_t;
fsm_state_t current_state, next_state;

// Instantiate the Topmodule
CNN_Accelerator_Top topmodule(
    .clk(wb_clk_i),
    .rst_n(wb_rst_i),
    .wb_adr_in(wb_adr_out),
    .data(data),
    .data_valid(data_valid),
    .data_out(data_out),
    .out_valid(out_valid),
    .full(full)
);

// Combinational Logic Block
always_comb begin
    // Default Values
    data_valid = 0;
    wb_adr_out = wbs_adr_i;
    data = wbs_dat_i;
    wbs_ack_o = 0;
    wbs_dat_o = 32'd0;
    next_state = current_state;
    
    case(current_state)
        IDLE: begin
            if(wbs_cyc_i && wbs_stb_i) begin
                if(wbs_we_i) begin
                    next_state = WRITE;
                end else begin
                    next_state = READ;
                end
            end
        end

        WRITE: begin
            data_valid = 1;
            if(full) begin
                // If topmodule is full, remain in WRITE state.
            end else begin
                next_state = IDLE;
                wbs_ack_o = 1;
            end
        end
        
        READ: begin
            if(out_valid) begin
                wbs_dat_o = data_out;
                wbs_ack_o = 1;
                next_state = IDLE;
            end
        end
        
        default: next_state = IDLE;
    endcase
end

// Sequential Logic Block
always_ff @(posedge wb_clk_i or negedge wb_rst_i) begin
    if(~wb_rst_i) begin
        current_state <= IDLE;
    end else begin
        current_state <= next_state;
    end
end

endmodule
```

### ChatGPT:
To add power pin declarations to the `Wishbone2Top` module, you would simply incorporate the provided power pins into the module declaration. Here's the updated `Wishbone2Top` module with the added power pin declarations:

```systemverilog
module Wishbone2Top(
    input wb_clk_i,
    input wb_rst_i,
    input wbs_stb_i,
    input wbs_cyc_i,
    input wbs_we_i,
    input [3:0] wbs_sel_i,
    input [31:0] wbs_dat_i,
    input [31:0] wbs_adr_i,
    output wbs_ack_o,
    output [31:0] wbs_dat_o,
    
    // Added power pins here
    `ifdef USE_POWER_PINS
        inout vccd1,	// User area 1 1.8V supply
        inout vssd1	// User area 1 digital ground
    `endif
);

// ... rest of the module remains unchanged ...
```

With this modification, when the `USE_POWER_PINS` compiler directive is defined, the power pins `vccd1` and `vssd1` will be added to the `Wishbone2Top` module's port list. If `USE_POWER_PINS` is not defined, these power pins will be ignored during compilation.
