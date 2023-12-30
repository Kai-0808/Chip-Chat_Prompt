## 12_Wishbone converter
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Wishbone Converter:

- Wishbone Interface:
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

**Functionality:**

Receive the Wishbone signals and convert them to the input signals of the Topmodule. Also, convert the output signals of the Topmodule to some of the input Wishbone signals.
The signals of the Topmodule are listed below. This is the only interface between the Topmodule and the Wishbone converter.
You can add any output signals to the converter if it's needed.
Separate the combinational and sequential logic block. Give the FSM and the entire SystemVerilog RTL code without any shortcut.

**Topmodule Signals:**
```systemverilog
module CNN_Accelerator_Top (
    input logic clk,                 // Clock signal
    input logic rst_n,               // Asynchronous active low reset signal
    input logic [31:0] wb_adr_in,    // The address of the instruction or the image pixel
    input logic [31:0] data,         // The instruction to be decoded or the image pixel to be loaded
    input logic data_valid,          // The signal to indicate the data is valid
    output logic [31:0] data_out,    // The data read from the memory
    output logic out_valid,          // The signal to indicate the validity of the data (used when opcode is Store)
    output logic full                // The signal to indicate the queues are full
);
    // Internal signals for interfacing Control Unit and Instruction Decoder
    logic [2:0] opcode;
    logic [4:0] saddr, daddr, map_size, sub_addr;
    logic [2:0] sub_size;
    logic stride;
    logic [1:0] padding;
    logic [31:0] wb_addr, data_in;
    logic [1:0] mem_type;
    logic done, instr_fault;

    // Instantiate Instruction Decoder
    instruction_decoder instr_decoder (
        .clk(clk),
        .rst_n(rst_n),
        .wb_adr_in(wb_adr_in),
        .data(data),
        .data_valid(data_valid),
        .done(done),
        .instr_fault(instr_fault),
        .opcode(opcode),
        .saddr(saddr),
        .daddr(daddr),
        .map_size(map_size),
        .sub_addr(sub_addr),
        .sub_size(sub_size),
        .stride(stride),
        .padding(padding),
        .wb_addr(wb_addr),
        .data_in(data_in),
        .mem_type(mem_type),
        .full(full)
    );

    // Instantiate Control Unit
    ControlUnit control_unit (
        .clk(clk),
        .rst_n(rst_n),
        .opcode(opcode),
        .saddr(saddr),
        .daddr(daddr),
        .map_size(map_size),
        .sub_addr(sub_addr),
        .sub_size(sub_size),
        .stride(stride),
        .padding(padding),
        .wb_addr(wb_addr),
        .data_in(data_in),
        .mem_type(mem_type),
        .data_out(data_out),
        .done(done),
        .out_valid(out_valid),
        .instr_fault(instr_fault)
    );

endmodule
```

### ChatGPT:
Here is the Wishbone to CNN_Accelerator_Top converter. I'll define the FSM in three states:

- `IDLE`: Wait for the incoming wishbone request.
- `WRITE`: Write data to the Top module.
- `READ`: Read data from the Top module.

### SystemVerilog RTL code:

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
    .rst_n(~wb_rst_i),
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

This design waits for a `wishbone` request (`wbs_cyc_i` and `wbs_stb_i` being high) and based on whether it's a write operation (`wbs_we_i` high) or read, it either pushes data into the `Topmodule` or reads data from it. 

For a write operation, `data_valid` is made high and data is pushed into the `Topmodule`. The design waits until the `Topmodule` is ready to accept more data (`full` signal low). After writing, it acknowledges the operation (`wbs_ack_o` high).

For a read operation, the design waits for `out_valid` to be high and then reads the data. Once data is read, it acknowledges the operation (`wbs_ack_o` high).

Note: The efficiency of this design might be improved depending on the specifics of the surrounding infrastructure and the expected performance constraints. Always test thoroughly in your desired setup.

### User:
Connect the wb_rst_i to rst of top directly

### ChatGPT:
Alright, let's modify the code to connect `wb_rst_i` directly to the `rst` of the `Topmodule`. This would mean that when `wb_rst_i` is asserted, the `Topmodule` will also get reset. 

Here's the modified portion:

```systemverilog
// Instantiate the Topmodule
CNN_Accelerator_Top topmodule(
    .clk(wb_clk_i),
    .rst_n(wb_rst_i),  // Direct connection
    .wb_adr_in(wb_adr_out),
    .data(data),
    .data_valid(data_valid),
    .data_out(data_out),
    .out_valid(out_valid),
    .full(full)
);
```

You don't need to make any other changes in the rest of the code. The `Topmodule` will now be reset whenever `wb_rst_i` is asserted.
