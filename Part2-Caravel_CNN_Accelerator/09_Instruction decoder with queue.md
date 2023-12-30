## 09_Instruction decoder with queue
### User:
Act as a professional hardware designer and SystemVerilog programmer. Design the module with the following spec.

### Instruction Decoder:

- Input:
  - clk: 1 bit. Clock signal.
  - rst_n: 1 bit. Asynchronous active low reset signal.
  - wb_adr_in: 32 bits. The address of the instruction or the image pixel.
  - data: 32 bits. The instruction to be decoded or the image pixel to be loaded.
  - data_valid: 1 bit. The signal to indicate the data is valid.
  - done: 1 bit. The signal from control unit to indicate the completion of the instruction.
  - instr_fault: 1 bit. The signal to indicate the fault of the instruction.

- Output:
  - opcode: 3 bits. The opcode of the instruction. 001 for Conv, 010 for Activation, 011 for Pooling, 100 for Bias, 101 for Load, 110 for Store.
  - saddr: 5 bits. The source address of the image in memory bank or register file to be read.
  - daddr: 5 bits. The destination address of the image in memory bank or register file to be written.
  - map_size: 5 bits. The size of the image.
  - sub_addr: 5 bits. When opcode is Conv, the address of the kernel. When opcode is Bias, the address of the bias.
  - sub_size: 3 bits. When opcode is Conv, the size of the kernel. When opcode is Pooling, the size of the window.
  - stride: 1 bit. The stride of the convolution or pooling. Only used when opcode is Conv or Pooling. 0 for stride 1, 1 for stride 2.
  - padding: 2 bits. The padding of the convolution. Only used when opcode is Conv. 00 for no padding, 01 for zero padding.
  - wb_addr: 32 bits. The address of the image in memory bank or register file to be written.
  - data_in: 32 bits. The data to be written. Only used when opcode is Load.
  - mem_type: 2 bits. The type of the memory. 00 for memory bank, 01 for kernel register, 10 for bias register. Only used when opcode is Load or Store.
  - full: 1 bit. The signal to indicate the queues are full.

**Functionality:**

There are two circular queues in the decoder to store data and addresses. The queue has a size of 8 (parameterized). Two pointers are used to indicate the head and tail of the queue. The head pointer points to the instruction to be decoded. The tail pointer points to the instruction to be stored. The queue is empty when the head pointer is equal to the tail pointer. The queue is full when the head pointer is one position ahead of the tail pointer.

The two queues may share the same pointers since they will work synchronously.

The data queue and addr queue push data and addr every cycle unless the queue is full.

If the addr[31:28] to push is 4'h3, then the data to push is an instruction. The corresponding output should be set based on the instruction. `data_in` should be set to 0.

Otherwise, if the addr[31:28] to push is not 4'h3, the data to push is a pixel, and a Load instruction should be issued. When Load is issued,  `map_size` should be set based on addr[18:17], 2'b00 is 31, 2'b01 is 6, and 2'b10 is 0; other fields should be set to 0. `data_in` should be set to the data to push.

The `wb_addr` is always set to the address to push.

The `full` signal should be set to 1 when the queues are full. If the queues are empty, the queue send all 0s as instr, `32'h3fff_ffff` to `wb_addr`, and all 0s to `data_in`.

There is no need for the FSM. Just operate the pointers like counters, and the queue like memory.

For other output signals, if the current instr does not use the signal, send a default value.

Separate the combinational and sequential logic block. Give the FSM and the entire SystemVerilog RTL code without any shortcut.

**ISA Table:**

| OpCode (Bin) | Instruction  | Field 1     | Field 2       | Field 3     | Field 4       | Field 5     | Field 6     | Field 7     | Field 8     |
|--------------|--------------|-------------|---------------|-------------|---------------|-------------|-------------|-------------|-------------|
| `001`       | Conv         | Saddr (5b)  | Daddr (5b)    | M. Size (5b)| Kernel Addr (5b) | K. Size (3b)| Stride (1b)| Padding (2b)| Reserved (3b)|
| `010`       | Acti         | Saddr (5b)  | Daddr (5b)    | -           | -             | -           | -           | -           | Reserved (19b)|
| `011`       | Pooling      | Saddr (5b)  | Daddr (5b)    | M. Size (5b)| -           | Window Size (3b)| W. Stride (1b)| -        Reserved (5b)|
| `100`       | Bias         | Saddr (5b)  | Daddr (5b)    | M. Size (5b)| Bias Addr (5b)| -           | -           | -           | Reserved (9b)|
| `101`       | Load         | -           | -           | M. Size (5b)| -           | -           | -           | -           | Reserved (14b)|
| `110`       | Store        | Saddr (5b)  | Daddr (5b)    | M. Size (5b)| Data Type (2b)| -           | -           | -           | Reserved (12b)|

Bit allocation (from the leftmost bit):

1. **OpCode**: 3 bits
2. **Saddr**: 5 bits
3. **Daddr**: 5 bits
4. **Map Size**: 5 bits
5. **Kernel/Window/Bias Addr/Data Type**: 5 bits (The 5th bit for Load/Store is used for the first bit of Data Type)
6. **Kernel/Window Size/Data Type**: 3 bits (The 3rd bit for Load/Store is used for the second bit of Data Type)
7. **Stride**: 1 bit
8. **Padding**: 2 bits (only for Conv)

### ChatGPT:
Sure, let's get started. 

**Plan**:
1. Create a module for the instruction decoder.
2. Declare the inputs and outputs.
3. Declare the circular queue and the head and tail pointers.
4. Decode the instruction based on the given ISA table.
5. Handle the "Load" instruction for the image pixel.
6. Create combinational logic for decoding and set the outputs accordingly.
7. Create sequential logic to update the pointers.

**RTL Code**:

```systemverilog
module instruction_decoder (
    input  wire         clk,
    input  wire         rst_n,
    input  wire [31:0]  wb_adr_in,
    input  wire [31:0]  data,
    input  wire         data_valid,
    input  wire         done,
    input  wire         instr_fault,
    output wire [2:0]   opcode,
    output wire [4:0]   saddr,
    output wire [4:0]   daddr,
    output wire [4:0]   map_size,
    output wire [4:0]   sub_addr,
    output wire [2:0]   sub_size,
    output wire         stride,
    output wire [1:0]   padding,
    output wire [31:0]  wb_addr,
    output wire [31:0]  data_in,
    output wire [1:0]   mem_type,
    output wire         full
);

    // Parameter for Queue Size
    parameter QUEUE_SIZE = 8;
    parameter POINTER_WIDTH = 3; // Since QUEUE_SIZE = 8, it needs 3 bits

    // Circular Queue for Data and Address
    reg [31:0] data_queue [0:QUEUE_SIZE-1];
    reg [31:0] addr_queue [0:QUEUE_SIZE-1];

    // Pointers for Head and Tail
    reg [POINTER_WIDTH-1:0] head_ptr = 0;
    reg [POINTER_WIDTH-1:0] tail_ptr = 0;

    // Temporary Variables
    wire [31:0] current_data;
    wire [31:0] current_addr;

    // Check if Queue is Full or Empty
    assign full = (head_ptr == tail_ptr + 1'b1) ? 1'b1 : 1'b0;
    wire empty = (head_ptr == tail_ptr) ? 1'b1 : 1'b0;

    // Combinational Logic Block
    always @* begin
        if (empty) begin
            opcode   = 3'b000;  // Default value
            saddr    = 5'b0;
            daddr    = 5'b0;
            map_size = 5'b0;
            sub_addr = 5'b0;
            sub_size = 3'b0;
            stride   = 1'b0;
            padding  = 2'b0;
            wb_addr  = 32'h3fff_ffff;
            data_in  = 32'b0;
            mem_type = 2'b00;
        end else if (current_addr[31:28] == 4'h3) begin
            opcode   = current_data[31:29];
            saddr    = current_data[28:24];
            daddr    = current_data[23:19];
            map_size = current_data[18:14];
            sub_addr = current_data[13:9];
            sub_size = current_data[8:6];
            stride   = current_data[5];
            padding  = current_data[4:3];
            wb_addr  = current_addr;
            data_in  = 32'b0;
            mem_type = 2'b00;
        end else begin // Load Instruction for Pixel
            opcode   = 3'b101;
            saddr    = 5'b0;
            daddr    = 5'b0;
            map_size = (current_addr[18:17] == 2'b00) ? 5'b11111 : (current_addr[18:17] == 2'b01) ? 5'b000110 : 5'b0;
            sub_addr = 5'b0;
            sub_size = 3'b0;
            stride   = 1'b0;
            padding  = 2'b0;
            wb_addr  = current_addr;
            data_in  = current_data;
            mem_type = 2'b00;
        end
    end

    // Sequential Logic Block
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            head_ptr <= 0;
            tail_ptr <= 0;
        end else if (data_valid && !full) begin
            addr_queue[tail_ptr] <= wb_adr_in;
            data_queue[tail_ptr] <= data;
            tail_ptr <= tail_ptr + 1'b1;
        end else if (done) begin
            head_ptr <= head_ptr + 1'b1;
        end
    end

    // Current data and address fetch
    assign current_data = data_queue[head_ptr];
    assign current_addr = addr_queue[head_ptr];

endmodule
```

This code provides the basic functionality of the instruction decoder as per the given specifications. It uses circular queues for data and address storage. The pointers are incremented using counters and the queue is operated like a memory. The decoder sets the outputs based on the instruction format provided in the ISA table. 

Note: This is a basic design. In an actual ASIC or FPGA implementation, there might be additional optimizations, error checks, and other features that might be necessary.

### User:
For `current_addr[31:28] == 4'h3`, `mem_type` needs to be assign according to `current_data[13:12]` when Store.

### ChatGPT:
Got it. Let's make that correction.

The modification will be in the combinational logic block. Specifically, when `current_addr[31:28] == 4'h3` and the instruction is `Store` (i.e., `opcode == 3'b110`), we'll use the bits `current_data[13:12]` to set the `mem_type`.

Here's the updated portion of the code:

```systemverilog
// Combinational Logic Block
always @* begin
    if (empty) begin
        // ... [same as before]
    end else if (current_addr[31:28] == 4'h3) begin
        opcode   = current_data[31:29];
        saddr    = current_data[28:24];
        daddr    = current_data[23:19];
        map_size = current_data[18:14];
        sub_addr = current_data[13:9];
        sub_size = current_data[8:6];
        stride   = current_data[5];
        padding  = current_data[4:3];
        wb_addr  = current_addr;
        data_in  = 32'b0;
        
        // Assign mem_type for Store instruction
        if (opcode == 3'b110) begin
            mem_type = current_data[13:12];
        end else begin
            mem_type = 2'b00;
        end
    end else begin // Load Instruction for Pixel
        // ... [same as before]
    end
end
```

This change ensures that `mem_type` is correctly assigned based on the `current_data[13:12]` bits when the instruction is `Store`.

### User:
Give the entire code again without shortcut.

### ChatGPT:
Of course! Here's the updated instruction decoder module in its entirety:

```systemverilog
module instruction_decoder (
    input  wire         clk,
    input  wire         rst_n,
    input  wire [31:0]  wb_adr_in,
    input  wire [31:0]  data,
    input  wire         data_valid,
    input  wire         done,
    input  wire         instr_fault,
    output wire [2:0]   opcode,
    output wire [4:0]   saddr,
    output wire [4:0]   daddr,
    output wire [4:0]   map_size,
    output wire [4:0]   sub_addr,
    output wire [2:0]   sub_size,
    output wire         stride,
    output wire [1:0]   padding,
    output wire [31:0]  wb_addr,
    output wire [31:0]  data_in,
    output wire [1:0]   mem_type,
    output wire         full
);

    // Parameter for Queue Size
    parameter QUEUE_SIZE = 8;
    parameter POINTER_WIDTH = 3; // Since QUEUE_SIZE = 8, it needs 3 bits

    // Circular Queue for Data and Address
    reg [31:0] data_queue [0:QUEUE_SIZE-1];
    reg [31:0] addr_queue [0:QUEUE_SIZE-1];

    // Pointers for Head and Tail
    reg [POINTER_WIDTH-1:0] head_ptr = 0;
    reg [POINTER_WIDTH-1:0] tail_ptr = 0;

    // Temporary Variables
    wire [31:0] current_data;
    wire [31:0] current_addr;

    // Check if Queue is Full or Empty
    assign full = (head_ptr == tail_ptr + 1'b1) ? 1'b1 : 1'b0;
    wire empty = (head_ptr == tail_ptr) ? 1'b1 : 1'b0;

    // Combinational Logic Block
    always @* begin
        if (empty) begin
            opcode   = 3'b000;
            saddr    = 5'b0;
            daddr    = 5'b0;
            map_size = 5'b0;
            sub_addr = 5'b0;
            sub_size = 3'b0;
            stride   = 1'b0;
            padding  = 2'b0;
            wb_addr  = 32'h3fff_ffff;
            data_in  = 32'b0;
            mem_type = 2'b00;
        end else if (current_addr[31:28] == 4'h3) begin
            opcode   = current_data[31:29];
            saddr    = current_data[28:24];
            daddr    = current_data[23:19];
            map_size = current_data[18:14];
            sub_addr = current_data[13:9];
            sub_size = current_data[8:6];
            stride   = current_data[5];
            padding  = current_data[4:3];
            wb_addr  = current_addr;
            data_in  = 32'b0;

            if (opcode == 3'b110) begin
                mem_type = current_data[13:12];
            end else begin
                mem_type = 2'b00;
            end
        end else begin
            opcode   = 3'b101;
            saddr    = 5'b0;
            daddr    = 5'b0;
            map_size = (current_addr[18:17] == 2'b00) ? 5'b11111 : 
                       (current_addr[18:17] == 2'b01) ? 5'b000110 : 5'b0;
            sub_addr = 5'b0;
            sub_size = 3'b0;
            stride   = 1'b0;
            padding  = 2'b0;
            wb_addr  = current_addr;
            data_in  = current_data;
            mem_type = 2'b00;
        end
    end

    // Sequential Logic Block
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            head_ptr <= 0;
            tail_ptr <= 0;
        end else if (data_valid && !full) begin
            addr_queue[tail_ptr] <= wb_adr_in;
            data_queue[tail_ptr] <= data;
            tail_ptr <= tail_ptr + 1'b1;
        end else if (done) begin
            head_ptr <= head_ptr + 1'b1;
        end
    end

    // Current data and address fetch
    assign current_data = data_queue[head_ptr];
    assign current_addr = addr_queue[head_ptr];

endmodule
```

I've integrated the modification for `mem_type` during `Store` operations into the entire module.

### User:
change the 5'b000110 to 5 BIT

### ChatGPT:
I apologize for the oversight. I understand that you'd like the value `5'b000110` for `map_size` to be represented in 5 bits. However, `5'b000110` is actually a 6-bit value. I assume there might have been a mistake in the earlier representation.

If I'm interpreting correctly, you'd like the value `6` to be represented in 5 bits. In binary, `6` is represented as `110`, which would be `5'b00110` in 5-bit representation.

Let's make that correction:

```systemverilog
map_size = (current_addr[18:17] == 2'b00) ? 5'b11111 : 
           (current_addr[18:17] == 2'b01) ? 5'b00110 : 5'b0;
```

This change corrects the representation of the value `6` in 5 bits for the `map_size` signal.

### User:
// Pointers for Head and Tail
    reg [POINTER_WIDTH-1:0] head_ptr = 0;
    reg [POINTER_WIDTH-1:0] tail_ptr = 0;
Do not assign value in the declaration.

### ChatGPT:
I understand. Let's initialize the pointers in the sequential logic block with the reset condition. Here's the corrected portion:

**Declaration**:
```systemverilog
// Pointers for Head and Tail
reg [POINTER_WIDTH-1:0] head_ptr;
reg [POINTER_WIDTH-1:0] tail_ptr;
```

**Initialization in Sequential Logic Block**:
```systemverilog
// Sequential Logic Block
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        head_ptr <= POINTER_WIDTH'd0;
        tail_ptr <= POINTER_WIDTH'd0;
    end else if (data_valid && !full) begin
        addr_queue[tail_ptr] <= wb_adr_in;
        data_queue[tail_ptr] <= data;
        tail_ptr <= tail_ptr + 1'b1;
    end else if (done) begin
        head_ptr <= head_ptr + 1'b1;
    end
end
```

This change ensures that the pointers are initialized only during a reset condition and not during the declaration.

### User:
Do the reset for both queue.

### ChatGPT:
Alright, let's also reset the `data_queue` and `addr_queue` during the reset condition. Here's the modification to the sequential logic block:

```systemverilog
// Sequential Logic Block
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        head_ptr <= POINTER_WIDTH'd0;
        tail_ptr <= POINTER_WIDTH'd0;
        
        // Reset the data and addr queues
        for (int i = 0; i < QUEUE_SIZE; i = i + 1) begin
            data_queue[i] <= 32'b0;
            addr_queue[i] <= 32'b0;
        end

    end else if (data_valid && !full) begin
        addr_queue[tail_ptr] <= wb_adr_in;
        data_queue[tail_ptr] <= data;
        tail_ptr <= tail_ptr + 1'b1;
    end else if (done) begin
        head_ptr <= head_ptr + 1'b1;
    end
end
```

The above change ensures that during a reset condition, both `data_queue` and `addr_queue` are cleared to all zeros across their entire lengths.

### User:
For the queues logic, it should always detect the conditions of `data_valid && !full` and `done` if not reset. Hence, they should be independent if-statement, not else-if-statement.
Put them in the else-statement of `!rst_n`.

### ChatGPT:
Understood. The logic should be structured such that during the reset condition, the queues and pointers are reset, and outside of the reset condition, the two conditions `data_valid && !full` and `done` are always checked.

Here's the corrected sequential logic block:

```systemverilog
// Sequential Logic Block
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        head_ptr <= POINTER_WIDTH'd0;
        tail_ptr <= POINTER_WIDTH'd0;
        
        // Reset the data and addr queues
        for (int i = 0; i < QUEUE_SIZE; i = i + 1) begin
            data_queue[i] <= 32'b0;
            addr_queue[i] <= 32'b0;
        end
    end else begin
        if (data_valid && !full) begin
            addr_queue[tail_ptr] <= wb_adr_in;
            data_queue[tail_ptr] <= data;
            tail_ptr <= tail_ptr + 1'b1;
        end

        if (done) begin
            head_ptr <= head_ptr + 1'b1;
        end
    end
end
```

This change ensures that both the `data_valid && !full` and `done` conditions are always checked outside the reset condition.
