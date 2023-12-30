## 15_user_proj_wrapper
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.
## To-Do:
Remove the `user_proj_example` in the `user_project_wrapper` module and replace it with the `Wishbone2Top` module.
### user_project_wrapper:
```systemverilog
module user_project_wrapper #(
    parameter BITS = 32
) (
`ifdef USE_POWER_PINS
    inout vdda1,	// User area 1 3.3V supply
    inout vdda2,	// User area 2 3.3V supply
    inout vssa1,	// User area 1 analog ground
    inout vssa2,	// User area 2 analog ground
    inout vccd1,	// User area 1 1.8V supply
    inout vccd2,	// User area 2 1.8v supply
    inout vssd1,	// User area 1 digital ground
    inout vssd2,	// User area 2 digital ground
`endif

    // Wishbone Slave ports (WB MI A)
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

    // Logic Analyzer Signals
    input  [127:0] la_data_in,
    output [127:0] la_data_out,
    input  [127:0] la_oenb,

    // IOs
    input  [`MPRJ_IO_PADS-1:0] io_in,
    output [`MPRJ_IO_PADS-1:0] io_out,
    output [`MPRJ_IO_PADS-1:0] io_oeb,

    // Analog (direct connection to GPIO pad---use with caution)
    // Note that analog I/O is not available on the 7 lowest-numbered
    // GPIO pads, and so the analog_io indexing is offset from the
    // GPIO indexing by 7 (also upper 2 GPIOs do not have analog_io).
    inout [`MPRJ_IO_PADS-10:0] analog_io,

    // Independent clock (on independent integer divider)
    input   user_clock2,

    // User maskable interrupt signals
    output [2:0] user_irq
);

/*--------------------------------------*/
/* User project is instantiated  here   */
/*--------------------------------------*/

user_proj_example mprj (
`ifdef USE_POWER_PINS
	.vccd1(vccd1),	// User area 1 1.8V power
	.vssd1(vssd1),	// User area 1 digital ground
`endif

    .wb_clk_i(wb_clk_i),
    .wb_rst_i(wb_rst_i),

    // MGMT SoC Wishbone Slave

    .wbs_cyc_i(wbs_cyc_i),
    .wbs_stb_i(wbs_stb_i),
    .wbs_we_i(wbs_we_i),
    .wbs_sel_i(wbs_sel_i),
    .wbs_adr_i(wbs_adr_i),
    .wbs_dat_i(wbs_dat_i),
    .wbs_ack_o(wbs_ack_o),
    .wbs_dat_o(wbs_dat_o),

    // Logic Analyzer

    .la_data_in(la_data_in),
    .la_data_out(la_data_out),
    .la_oenb (la_oenb),

    // IO Pads

    .io_in ({io_in[37:30],io_in[7:0]}),
    .io_out({io_out[37:30],io_out[7:0]}),
    .io_oeb({io_oeb[37:30],io_oeb[7:0]}),

    // IRQ
    .irq(user_irq)
);

endmodule	// user_project_wrapper

`default_nettype wire
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
    output [31:0] wbs_dat_o,
    
    // Added power pins here
    `ifdef USE_POWER_PINS
        inout vccd1,	// User area 1 1.8V supply
        inout vssd1	// User area 1 digital ground
    `endif
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
Based on the given specifications, I will integrate the `Wishbone2Top` module into the `user_project_wrapper` module by replacing the `user_proj_example` instantiation. The steps are as follows:

1. Include the USE_POWER_PINS conditional compilation check in the `Wishbone2Top` module.
2. Modify the `user_project_wrapper` to instantiate the `Wishbone2Top` module.
3. Pass all necessary signals and ports to the `Wishbone2Top` module.

Here's the modified `user_project_wrapper` with the `Wishbone2Top` module integrated:

```systemverilog
module user_project_wrapper #(
    parameter BITS = 32
) (
`ifdef USE_POWER_PINS
    inout vdda1,    // User area 1 3.3V supply
    inout vdda2,    // User area 2 3.3V supply
    inout vssa1,    // User area 1 analog ground
    inout vssa2,    // User area 2 analog ground
    inout vccd1,    // User area 1 1.8V supply
    inout vccd2,    // User area 2 1.8v supply
    inout vssd1,    // User area 1 digital ground
    inout vssd2,    // User area 2 digital ground
`endif

    // Wishbone Slave ports (WB MI A)
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

    // Logic Analyzer Signals
    input  [127:0] la_data_in,
    output [127:0] la_data_out,
    input  [127:0] la_oenb,

    // IOs
    input  [`MPRJ_IO_PADS-1:0] io_in,
    output [`MPRJ_IO_PADS-1:0] io_out,
    output [`MPRJ_IO_PADS-1:0] io_oeb,

    // Analog (direct connection to GPIO pad---use with caution)
    inout [`MPRJ_IO_PADS-10:0] analog_io,

    // Independent clock (on independent integer divider)
    input   user_clock2,

    // User maskable interrupt signals
    output [2:0] user_irq
);

/*--------------------------------------*/
/* User project is instantiated here    */
/*--------------------------------------*/

Wishbone2Top mprj (
`ifdef USE_POWER_PINS
    .vccd1(vccd1),  // User area 1 1.8V power
    .vssd1(vssd1),  // User area 1 digital ground
`endif

    .wb_clk_i(wb_clk_i),
    .wb_rst_i(wb_rst_i),
    .wbs_cyc_i(wbs_cyc_i),
    .wbs_stb_i(wbs_stb_i),
    .wbs_we_i(wbs_we_i),
    .wbs_sel_i(wbs_sel_i),
    .wbs_adr_i(wbs_adr_i),
    .wbs_dat_i(wbs_dat_i),
    .wbs_ack_o(wbs_ack_o),
    .wbs_dat_o(wbs_dat_o)
);

endmodule  // user_project_wrapper

`default_nettype wire
```

This modification assumes that the `Wishbone2Top` module handles all necessary internal functions without requiring additional signals from the `user_project_wrapper`. Ensure that all the required signals, especially those related to IO pads, logic analyzer, and other functionalities, are properly interfaced if necessary.
