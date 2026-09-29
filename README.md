Traffic Light Controller
A finite state machine on the Nexys A7 FPGA
FPGA for DSP Applications  |  Verilog  |  Vivado
1. Overview
This project implements a traffic light controller as a finite state machine (FSM) in Verilog and runs it on a Digilent Nexys A7-100T FPGA board. The light cycles through three states on its own: Green for 10 seconds, Blue for 3 seconds and Red for 7 seconds. An RGB LED shows the current color, and one digit of the 7-segment display shows a countdown of the seconds left in that color. Pressing the reset button returns the controller to Green from any state.
Key features
•	Three-state FSM with unequal, realistic durations
•	The 100 MHz board clock is divided down to a precise one-second pulse
•	One counter (elapsed_sec) is reused for all three states
•	Countdown shown on a single 7-segment digit, so no display multiplexing is needed
•	Active-low reset that returns the design to Green
2. Hardware and tools
Item	Detail
Board	Digilent Nexys A7-100T
FPGA	Xilinx Artix-7 XC7A100T (part xc7a100tcsg324-1)
Clock	100 MHz onboard oscillator
Input	CPU RESET button (active-low)
Outputs	RGB LED LD16, and the rightmost digit of the 7-segment display
Language	Verilog
Tools	Xilinx Vivado: synthesis, implementation, bitstream generation and board programming
3. Specification
State	Duration	RGB LED	Digit shown (countdown)
Green	10 s	Green	9 → 0
Blue	3 s	Blue	2 → 0
Red	7 s	Red	6 → 0

•	After Red, the FSM returns to Green by itself, so the cycle repeats forever.
•	Pressing reset in any state forces Green and restarts the countdown at 9.
4. Design architecture
The design has three parts, all driven by the same 100 MHz clock:
100 MHz clock
board oscillator	→	Clock divider
tick_1hz pulse	→	FSM
state, elapsed_sec	→	Outputs
RGB LED + digit

•	Clock divider (top module): counts 100,000,000 clock ticks and produces one tick_1hz pulse.
•	FSM (top module): on each tick_1hz, counts seconds in the current color and moves to the next state when that color's time is up.
•	Outputs: three comparisons on state drive the RGB LED, and the module seg7_countdown turns state and elapsed_sec into the digit.
Source files: top_traffic_light.v, seg7_countdown.v and the constraints file top_traffic_light.xdc (Appendix A to C).
Key signals
Signal	Bits	Purpose
CLK100MHZ	1	100 MHz board clock
CPU_RESETN	1	Reset button, active-low (0 means pressed)
tick_counter	27	Counts clock ticks up to 99,999,999
tick_1hz	1	High for one clock cycle once per second
state	2	Current color: 0 = Green, 1 = Blue, 2 = Red
elapsed_sec	4	Seconds passed in the current color; counts up from 0
target_max	4	Current color's duration minus 1 (9, 2 or 6); in seg7_countdown
remaining	4	target_max minus elapsed_sec; the digit that is displayed
5. How it works
5.1 Clock divider
The board clock ticks 100,000,000 times per second, which is far too fast to time a traffic light directly. tick_counter counts from 0 up to 99,999,999, which is exactly 100,000,000 ticks. When it reaches 99,999,999, tick_1hz is 1 for a single clock cycle and the counter restarts from 0. That pulse is the one-second heartbeat for the whole design.
The register is 27 bits wide because 26 bits only reach 67,108,863, which is too small, while 27 bits reach 134,217,727.
5.2 State machine
The FSM block runs on every clock edge, but it only makes decisions when tick_1hz is 1, so effectively once per second. In each state it checks whether that color's time is up. If so, it moves to the next state and sets elapsed_sec back to 0. If not, it adds one second. Red leads back to Green, which makes the cycle repeat.
Example: Green lasts 10 ticks. elapsed_sec starts at 0, so the ten values 0 to 9 make ten seconds, which is why the code compares against GREEN_TIME - 1.
Tick	elapsed_sec before	Action
1st	0	Not 9, so elapsed_sec becomes 1
2nd	1	Not 9, so elapsed_sec becomes 2
...	...	...
9th	8	Not 9, so elapsed_sec becomes 9
10th	9	Equals GREEN_TIME - 1: state becomes Blue, elapsed_sec becomes 0

Blue works the same way with BLUE_TIME - 1 = 2 (3 ticks), and Red with RED_TIME - 1 = 6 (7 ticks).
5.3 Countdown display
elapsed_sec always counts up. The display module subtracts it from the current color's maximum (target_max), which turns the up-counter into a countdown: remaining = target_max - elapsed_sec.
elapsed_sec	Green shows	Blue shows	Red shows
0	9	2	6
1	8	1	5
2	7	0	4
3	6	-	3
4	5	-	2
5	4	-	1
6	3	-	0
7	2	-	-
8	1	-	-
9	0	-	-
5.4 From a number to lit segments
Each 7-segment digit has seven segments: a (top), b (upper right), c (lower right), d (bottom), e (lower left), f (upper left) and g (middle). A case statement works as a lookup table that turns the number 0 to 9 into the set of segments to light. The pattern is written in the order a, b, c, d, e, f, g, with 1 meaning on.
Digit	Segments lit	Pattern (a b c d e f g)
0	a b c d e f	1111110
1	b c	0110000
2	a b d e g	1101101
3	a b c d g	1111001
4	b c f g	0110011
5	a c d f g	1011011
6	a c d e f g	1011111
7	a b c	1110000
8	a b c d e f g	1111111
9	a b c d f g	1111011

The board's segments and digit enables are active-low (0 means on). So the code inverts the pattern (~pattern) before it reaches the pins, and AN = 8'b11111110 enables only the rightmost digit.
6. Code walkthrough
6.1 top_traffic_light.v
Code	What it does
parameter TICK_DIV = 100_000_000	Clock ticks per second. It is a parameter so a testbench can shorten it for simulation.
localparam S_GREEN / S_BLUE / S_RED	Named 2-bit state values. localparam because they are internal constants that should not be overridden from outside.
GREEN_TIME / BLUE_TIME / RED_TIME	Duration of each state in seconds (10, 3, 7).
reg state, elapsed_sec, tick_counter	The design's memory: current color, seconds in this color, and ticks toward the next second.
wire tick_1hz	A live comparison that is true for one clock cycle when tick_counter reaches 99,999,999.
First always block	Clock divider. Reset, or reaching a full second, clears the counter; otherwise it adds one.
Second always block	The FSM. Reset forces Green. Otherwise, on tick_1hz only: if the color's time is up, go to the next state and clear elapsed_sec; if not, add one second.
default branch	Catches the unused fourth value of state and returns to Green.
assign LED16_*	One comparison per color, so exactly one color is on at a time.
seg7_countdown disp (...)	Instantiates the display module and connects state and elapsed_sec to it.
6.2 seg7_countdown.v
Code	What it does
target_max	The current color's duration minus 1: 9, 2 or 6.
remaining = target_max - elapsed_sec	The countdown value shown on the digit.
case (remaining)	Lookup table from the number 0 to 9 to the segment pattern. default turns all segments off.
assign AN = 8'b11111110	Enables only the rightmost digit (active-low).
assign {CA, ..., CG} = ~pattern	Inverts the pattern for the active-low segments.
assign DP = 1'b1	Holds the decimal point off.
7. Verification
The FSM was simulated with TICK_DIV reduced from 100,000,000 to 20, so one simulated second takes 20 clock cycles and a full cycle runs quickly. The table shows the state, LED and digit sampled once per simulated second.
Second	State	RGB LED	Digit shown
0	Green	Green on	9
1	Green	Green on	8
2	Green	Green on	7
3	Green	Green on	6
4	Green	Green on	5
5	Green	Green on	4
6	Green	Green on	3
7	Green	Green on	2
8	Green	Green on	1
9	Green	Green on	0
10	Blue	Blue on	2
11	Blue	Blue on	1
12	Blue	Blue on	0
13	Red	Red on	6
14	Red	Red on	5
15	Red	Red on	4
16	Red	Red on	3
17	Red	Red on	2
18	Red	Red on	1
19	Red	Red on	0

•	Green lasts 10 seconds (0 to 9), Blue 3 seconds (10 to 12) and Red 7 seconds (13 to 19), then the FSM returns to Green at second 20.
•	The digit counts 9 → 0 for Green, 2 → 0 for Blue and 6 → 0 for Red, as specified.
•	Only AN[0] is enabled (11111110) in every sample, so only the rightmost digit is lit.
•	A reset in the middle of a cycle returned the FSM to Green in simulation.
•	The design was then programmed onto the Nexys A7 and observed cycling as specified.
8. Pin assignments
Signal	FPGA pin	Board resource
CLK100MHZ	E3	100 MHz clock
CPU_RESETN	C12	CPU RESET button (active-low)
LED16_R	N15	RGB LED LD16, red
LED16_G	M16	RGB LED LD16, green
LED16_B	R12	RGB LED LD16, blue
CA to CG	T10, R10, K16, K13, P15, T11, L18	7-segment segments a to g (active-low)
DP	H15	Decimal point
AN[0] to AN[7]	J17, J18, T9, J14, P14, T14, K2, U13	Digit enables (active-low)
9. Demo procedure
1.	Program the board and confirm that only the rightmost 7-segment digit is lit.
2.	Press reset. The RGB LED is green and the digit shows 9.
3.	Watch Green count 9 → 0 over 10 seconds.
4.	At 0 the LED turns blue and the digit restarts at 2. Blue lasts 3 seconds.
5.	The LED then turns red and the digit restarts at 6. Red lasts 7 seconds.
6.	After Red, the light returns to Green by itself, with no button press.
7.	Press reset during Blue or Red. The controller returns to Green and 9 immediately.
10. Likely questions
Question	Answer
Why does a 10-second Green count 0 to 9?	elapsed_sec starts at 0, so 0 through 9 is ten values, which is ten seconds. The display shows the countdown 9 → 0.
Why compare with TICK_DIV - 1?	The counter starts at 0, so counting 0 to 99,999,999 is exactly 100,000,000 ticks.
Why 27 bits for tick_counter?	It must reach 99,999,999. 26 bits hold up to 67,108,863 (too small); 27 bits hold up to 134,217,727.
What is an FSM?	A system with a fixed set of states and rules for moving between them. Here the states are Green, Blue and Red, and a timer decides when to move.
Why localparam and not parameter?	The state codes and durations are internal facts of the design, so localparam stops them being overridden from outside. TICK_DIV is a parameter because it is meant to be changed for simulation.
Why does the FSM only act on tick_1hz?	The block is clocked at 100 MHz, but the if (tick_1hz) check means it only makes decisions once per second. On other clock cycles it does nothing.
Why Blue instead of Yellow?	Blue is easy to see on the RGB LED and needs only one color pin.
What does ~pattern do?	The board's segments are active-low, so the inversion turns the "on" bits into 0s.
Appendix A: top_traffic_light.v
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////
// top_traffic_light.v
// Traffic light FSM: GREEN (10 s) -> BLUE (3 s) -> RED (7 s) -> repeat.
// CPU_RESETN returns the design to GREEN from any state.
//////////////////////////////////////////////////////////////////////////////
 
module top_traffic_light #(
    // 100 MHz clock ticks that make up one real second
    parameter integer TICK_DIV = 100_000_000
)(
    input  wire        CLK100MHZ,
    input  wire        CPU_RESETN,
    output wire        LED16_R, LED16_G, LED16_B,
    output wire [7:0]  AN,
    output wire        CA, CB, CC, CD, CE, CF, CG,
    output wire        DP
);
 
    // FSM states (2 bits): 0 = Green, 1 = Blue, 2 = Red
    localparam [1:0] S_GREEN = 2'd0;
    localparam [1:0] S_BLUE  = 2'd1;
    localparam [1:0] S_RED   = 2'd2;
 
    // Duration of each state, in seconds
    localparam [3:0] GREEN_TIME = 4'd10;
    localparam [3:0] BLUE_TIME  = 4'd3;
    localparam [3:0] RED_TIME   = 4'd7;
 
    // Which color the light is in, starting at Green
    reg [1:0]  state        = S_GREEN;
    // Seconds passed in the current color, starting at 0
    reg [3:0]  elapsed_sec  = 4'd0;
    // Clock ticks counted toward the next full second
    reg [26:0] tick_counter = 27'd0;
    // High for one clock cycle when a full second has been counted
    wire       tick_1hz     = (tick_counter == TICK_DIV - 1);
 
    // Clock divider: 100 million ticks -> one tick_1hz pulse
    always @(posedge CLK100MHZ or negedge CPU_RESETN) begin
        if (!CPU_RESETN)
            tick_counter <= 27'd0;      // reset button pressed
        else if (tick_1hz)
            tick_counter <= 27'd0;      // one second reached, restart
        else
            tick_counter <= tick_counter + 1'b1;
    end
 
    // FSM: decisions are made once per second, on tick_1hz
    always @(posedge CLK100MHZ or negedge CPU_RESETN) begin
        if (!CPU_RESETN) begin
            state       <= S_GREEN;     // back to Green
            elapsed_sec <= 4'd0;        // counter back to 0
        end else if (tick_1hz) begin
            case (state)
                // Green ends on the 10th tick (elapsed_sec == 9)
                S_GREEN: if (elapsed_sec == GREEN_TIME - 1) begin
                             state <= S_BLUE; elapsed_sec <= 4'd0;
                         end else begin
                             elapsed_sec <= elapsed_sec + 1'b1;
                         end
                // Blue ends on the 3rd tick (elapsed_sec == 2)
                S_BLUE:  if (elapsed_sec == BLUE_TIME - 1) begin
                             state <= S_RED; elapsed_sec <= 4'd0;
                         end else begin
                             elapsed_sec <= elapsed_sec + 1'b1;
                         end
                // Red ends on the 7th tick (elapsed_sec == 6)
                S_RED:   if (elapsed_sec == RED_TIME - 1) begin
                             state <= S_GREEN; elapsed_sec <= 4'd0;
                         end else begin
                             elapsed_sec <= elapsed_sec + 1'b1;
                         end
                // Unused 4th state value: recover to Green
                default: begin
                             state <= S_GREEN; elapsed_sec <= 4'd0;
                         end
            endcase
        end
    end
 
    // RGB LED: exactly one color is on at a time
    assign LED16_G = (state == S_GREEN);
    assign LED16_B = (state == S_BLUE);
    assign LED16_R = (state == S_RED);
 
    // Countdown digit on the 7-segment display
    seg7_countdown disp (
        .state       (state),
        .elapsed_sec (elapsed_sec),
        .AN          (AN),
        .CA (CA), .CB (CB), .CC (CC), .CD (CD),
        .CE (CE), .CF (CF), .CG (CG),
        .DP          (DP)
    );
 
endmodule
Appendix B: seg7_countdown.v
// =============================================================================
// seg7_countdown.v
//
// Shows the seconds remaining in the current traffic light state on ONE
// 7-segment digit, counting down to 0 right as the state switches.
// Pure combinational logic -- reuses elapsed_sec from the FSM, no new
// counter, no clock, no multiplexing (single digit, standard 0-9 shapes).
// =============================================================================
 
module seg7_countdown (
    input  wire [1:0]  state,        // 0=GREEN, 1=BLUE, 2=RED
    input  wire [3:0]  elapsed_sec,  // seconds elapsed in the current state
    output wire [7:0]  AN,
    output wire         CA, CB, CC, CD, CE, CF, CG,
    output wire         DP
);
 
    localparam [1:0] S_GREEN = 2'd0;
    localparam [1:0] S_BLUE  = 2'd1;
    localparam [1:0] S_RED   = 2'd2;
 
    localparam [3:0] GREEN_TIME = 4'd10;
    localparam [3:0] BLUE_TIME  = 4'd3;
    localparam [3:0] RED_TIME   = 4'd7;
 
    // Total duration of whichever state is currently active, minus 1
    wire [3:0] target_max = (state == S_GREEN) ? (GREEN_TIME - 1) :
                             (state == S_BLUE)  ? (BLUE_TIME  - 1) :
                                                   (RED_TIME   - 1);
 
    // Counts down: 9,8,...,0 for green; 2,1,0 for blue; 6,...,0 for red
    wire [3:0] remaining = target_max - elapsed_sec;
 
    // Standard 7-segment digit patterns, bit order {a,b,c,d,e,f,g}, 1=on
    reg [6:0] pattern;
    always @(*) begin
        case (remaining)
            4'd0: pattern = 7'b1111110;
            4'd1: pattern = 7'b0110000;
            4'd2: pattern = 7'b1101101;
            4'd3: pattern = 7'b1111001;
            4'd4: pattern = 7'b0110011;
            4'd5: pattern = 7'b1011011;
            4'd6: pattern = 7'b1011111;
            4'd7: pattern = 7'b1110000;
            4'd8: pattern = 7'b1111111;
            4'd9: pattern = 7'b1111011;
            default: pattern = 7'b0000000;
        endcase
    end
 
    assign AN = 8'b11111110; // only the rightmost digit enabled
    assign {CA, CB, CC, CD, CE, CF, CG} = ~pattern;
    assign DP = 1'b1;
 
endmodule
Appendix C: top_traffic_light.xdc
## Pins from Digilent's Nexys-A7-100T-Master.xdc
 
## Clock (100 MHz)
set_property -dict { PACKAGE_PIN E3  IOSTANDARD LVCMOS33 } [get_ports { CLK100MHZ }];
create_clock -add -name sys_clk_pin -period 10.00 -waveform {0 5} [get_ports { CLK100MHZ }];
 
## CPU reset button (active-low)
set_property -dict { PACKAGE_PIN C12 IOSTANDARD LVCMOS33 } [get_ports { CPU_RESETN }];
 
## RGB LED 16
set_property -dict { PACKAGE_PIN N15 IOSTANDARD LVCMOS33 } [get_ports { LED16_R }];
set_property -dict { PACKAGE_PIN M16 IOSTANDARD LVCMOS33 } [get_ports { LED16_G }];
set_property -dict { PACKAGE_PIN R12 IOSTANDARD LVCMOS33 } [get_ports { LED16_B }];
 
## 7-segment display: segments and decimal point
set_property -dict { PACKAGE_PIN T10 IOSTANDARD LVCMOS33 } [get_ports { CA }];
set_property -dict { PACKAGE_PIN R10 IOSTANDARD LVCMOS33 } [get_ports { CB }];
set_property -dict { PACKAGE_PIN K16 IOSTANDARD LVCMOS33 } [get_ports { CC }];
set_property -dict { PACKAGE_PIN K13 IOSTANDARD LVCMOS33 } [get_ports { CD }];
set_property -dict { PACKAGE_PIN P15 IOSTANDARD LVCMOS33 } [get_ports { CE }];
set_property -dict { PACKAGE_PIN T11 IOSTANDARD LVCMOS33 } [get_ports { CF }];
set_property -dict { PACKAGE_PIN L18 IOSTANDARD LVCMOS33 } [get_ports { CG }];
set_property -dict { PACKAGE_PIN H15 IOSTANDARD LVCMOS33 } [get_ports { DP }];
 
## 7-segment display: digit enables
set_property -dict { PACKAGE_PIN J17 IOSTANDARD LVCMOS33 } [get_ports { AN[0] }];
set_property -dict { PACKAGE_PIN J18 IOSTANDARD LVCMOS33 } [get_ports { AN[1] }];
set_property -dict { PACKAGE_PIN T9  IOSTANDARD LVCMOS33 } [get_ports { AN[2] }];
set_property -dict { PACKAGE_PIN J14 IOSTANDARD LVCMOS33 } [get_ports { AN[3] }];
set_property -dict { PACKAGE_PIN P14 IOSTANDARD LVCMOS33 } [get_ports { AN[4] }];
set_property -dict { PACKAGE_PIN T14 IOSTANDARD LVCMOS33 } [get_ports { AN[5] }];
set_property -dict { PACKAGE_PIN K2  IOSTANDARD LVCMOS33 } [get_ports { AN[6] }];
set_property -dict { PACKAGE_PIN U13 IOSTANDARD LVCMOS33 } [get_ports { AN[7] }];
