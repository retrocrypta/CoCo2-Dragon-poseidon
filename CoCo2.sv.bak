//============================================================================
//
//  This program is free software; you can redistribute it and/or modify it
//  under the terms of the GNU General Public License as published by the Free
//  Software Foundation; either version 2 of the License, or (at your option)
//  any later version.
//
//  This program is distributed in the hope that it will be useful, but WITHOUT
//  ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
//  FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for
//  more details.
//
//  You should have received a copy of the GNU General Public License along
//  with this program; if not, write to the Free Software Foundation, Inc.,
//  51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
//============================================================================


module CoCo2
(       
        output        LED,                                              
        output        VGA_HS,
        output        VGA_VS,
        output        AUDIO_L,
        output        AUDIO_R, 
		  output [15:0]  DAC_L, 
		  output [15:0]  DAC_R, 
        input         TAPE_IN,
        input         UART_RXD,
        output        UART_TXD,
        input         SPI_SCK,
        output        SPI_DO,
        input         SPI_DI,
        input         SPI_SS2,
        input         SPI_SS3,
        input         CONF_DATA0,
        input         CLOCK_27,
        output  [5:0] VGA_R,
        output  [5:0] VGA_G,
        output  [5:0] VGA_B,

		  output [12:0] SDRAM_A,
		  inout  [15:0] SDRAM_DQ,
		  output        SDRAM_DQML,
        output        SDRAM_DQMH,
        output        SDRAM_nWE,
        output        SDRAM_nCAS,
        output        SDRAM_nRAS,
        output        SDRAM_nCS,
        output  [1:0] SDRAM_BA,
        output        SDRAM_CLK,
        output        SDRAM_CKE
);


 
assign LED  =  tape_in;


`include "build_id.v"
localparam CONF_STR = {
	"CoCo2;;",
	"F,CCCROM,Load Cartridge;",
	"OC,Tape Input,File,Line;",
	"F,CAS,Load Cassette;",
	"TF,Stop & Rewind;",
	"OD,Monitor Tape Sound,No,Yes;",
	"P1,Video Settings;",
	"P1OFH,Scandoubler Fx,None,HQ2x,CRT 25%,CRT 50%,CRT 75%;", 
	"P1O4,Overscan,Hidden,Visible;",
	"P1O3,Artifact,Enable,Disable;",
	"P1O2,Artifact Phase,Normal,Reverse;",
	"OM,Keyboard Layout,PC,CoCo;",
	"OL,D-Pad Joystick emu,No,Yes;",
	"OA,Swap Joysticks,Off,On;",
	"O89,Machine,CoCo2,Dragon32,Dragon64;",
	"O7,Turbo,Off,On;",
	"T0,Reset;",
	"V,v",`BUILD_DATE
};


/////////////////  CLOCKS  ////////////////////////

wire clk_sys;
wire locked;

pll pll
(
	.inclk0(CLOCK_27),
	.c0(clk_sys),
	.locked(locked)
);




/////////////////  HPS  ///////////////////////////


wire forced_scandoubler;
wire  [1:0] buttons;
wire [31:0] status;
wire [10:0] ps2_key;
wire        ioctl_download;
wire        ioctl_wr;
wire [15:0] ioctl_addr;
wire  [7:0] ioctl_data;
wire  [7:0] ioctl_index;

wire [31:0] joy1, joy2;
wire [15:0] joya1, joya2;

wire ypbpr;



 
mist_io #(.STRLEN($size(CONF_STR)>>3),.PS2DIV(100)) mist_io
(
	.SPI_SCK   (SPI_SCK),
   .CONF_DATA0(CONF_DATA0),
   .SPI_SS2   (SPI_SS2),
   .SPI_DO    (SPI_DO),
   .SPI_DI    (SPI_DI),

	.clk_sys(clk_sys),
	.conf_str(CONF_STR),

	.buttons(buttons),
	.status(status),
	.scandoubler_disable(forced_scandoubler),
   .ypbpr     (ypbpr),
	
	.ioctl_ce(1),
	.ioctl_download(ioctl_download),
	.ioctl_index(ioctl_index),
	.ioctl_wr(ioctl_wr),
	.ioctl_addr(ioctl_addr),
	.ioctl_dout(ioctl_data),
		
	
	.ps2_key(ps2_key),
	.joystick_analog_0(joya1),
   .joystick_analog_1(joya2),

	.joystick_0(joy1), // HPS joy [4:0] {Fire, Up, Down, Left, Right}
	.joystick_1(joy2)

);

/////////////////  RESET  /////////////////////////

wire reset = status[0] | buttons[1] | (ioctl_download && ioctl_index==1) | machine_select_reset;



////////////////  Console  ////////////////////////

wire sndout;
wire [15:0] sound_pad =  {sndout,sound[11:6], sound[5] ^ (status[13] ? (status[12] ? tape_in : casdout) : 1'b0), sound[4:0], 3'b0};

dac_audio #(16) dac_l (
   .clk_i        (clk_sys),
   .res_n_i      (1      ),
   .dac_i        (sound_pad),
   .dac_o        (AUDIO_L)
);

assign DAC_L=sound_pad;
assign DAC_R=sound_pad;
assign AUDIO_R=AUDIO_L;

wire HBlank;
wire HSync;
wire VBlank;
wire VSync;

wire [7:0] red;
wire [7:0] green;
wire [7:0] blue;

wire [11:0] sound;


wire [9:0] center_joystick_y1   =  8'd128 + joya1[15:8];
wire [9:0] center_joystick_x1   =  8'd128 + joya1[7:0];
wire [9:0] center_joystick_y2   =  8'd128 + joya2[15:8];
wire [9:0] center_joystick_x2   =  8'd128 + joya2[7:0];
wire vclk;

wire [31:0] coco_joy1 = status[10] ? joy2 : joy1;
wire [31:0] coco_joy2 = status[10] ? joy1 : joy2;

wire [15:0] coco_ajoy1 = status[10] ? {center_joystick_x2[7:0],center_joystick_y2[7:0]} : {center_joystick_x1[7:0],center_joystick_y1[7:0]};
wire [15:0] coco_ajoy2 = status[10] ? {center_joystick_x1[7:0],center_joystick_y1[7:0]} : {center_joystick_x2[7:0],center_joystick_y2[7:0]};

wire casdout;
wire cas_relay;

reg dragon64;
reg dragon;
reg [1:0]machineselect_r;
reg machine_select_reset;
reg [3:0]reset_count;

always @(posedge clk_sys)
begin
 machine_select_reset <=1'b0;
 dragon64 <= (status[9:8]==2'b10);
 dragon   <= (status[9:8]!=2'b00);
 
 if (machineselect_r!=status[9:8])
	reset_count<=4'b1111;

 if (reset_count>4'b0) begin
  reset_count<=reset_count - 4'b0001;
  machine_select_reset<=1'b1;
 end
 
 machineselect_r<=status[9:8];
end


dragoncoco dragoncoco(
  .clk(clk_sys), // 50 mhz
  .turbo(status[7]&cas_relay),
  .reset(~reset),
  .dragon(dragon),
  .dragon64(dragon64),
  .kblayout(~status[22]),
  .red(red),
  .green(green),
  .blue(blue),

  .hblank(HBlank),
  .vblank(VBlank),
  .hsync(HSync),
  .vsync(VSync),
  .vclk(vclk),
  // input ps2_clk,
  // input ps2_dat,
  .uart_din(1'b0),

  .ps2_key(ps2_key),
  .ioctl_addr(ioctl_addr),
  .ioctl_data(ioctl_data),
  .ioctl_download(ioctl_download),
  .ioctl_index(ioctl_index),
  .ioctl_wr(ioctl_wr),
  .casdout(status[12] ? tape_in : casdout),
  .cas_relay(cas_relay),
  .artifact_phase(status[2]),
  .artifact_enable(~status[3]),
  .overscan(status[4]),
//  .count_offset(status[8:5]),

  .joy_use_dpad(status[21]),

  .joy1(coco_joy1),
  .joy2(coco_joy2),

  .joya1(coco_ajoy1),
  .joya2(coco_ajoy2),


  .cass_snd(),
  .sound(sound),
  .sndout(sndout),

  .v_count(VCount),
  .h_count(HCount),
  .DLine1(DLine1),
  .DLine2(DLine2),

  .clk_Q_out(clk_Q_out)

);

reg [8:0] HCount,VCount;
wire [159:0]DLine1;
wire [159:0]DLine2;

wire clk_Q_out;

wire [24:0] sdram_addr;
wire [7:0] sdram_data;
wire sdram_rd;
wire load_tape = ioctl_index == 2;

sdram sdram
(
	.*,
	.init(~locked),
	.clk(clk_sys),
	.addr(ioctl_download ? ioctl_addr : sdram_addr),
	.wtbt(0),
	.dout(sdram_data),
	.din(ioctl_data),
	.rd(sdram_rd),
	.we(ioctl_wr & load_tape),
	.ready()
);

cassette cassette(
  .clk(clk_sys),
  .Q(clk_Q_out),

  .rewind(status[15]),
  .en(cas_relay),

  .sdram_addr(sdram_addr),
  .sdram_data(sdram_data),
  .sdram_rd(sdram_rd),

  .data(casdout)
//   .status(tape_status)
);

/////////////////  VIDEO  /////////////////////////


wire [2:0] scale = status[17:15];
wire       scandoubler = (scale || forced_scandoubler);

wire [7:0]rr = red;
wire [7:0]gg = green;
wire [7:0]bb = blue;
	
video_mixer #(.LINE_LENGTH(380)) video_mixer
(
   .*,
   .ce_pix(vclk),
   .ce_pix_actual(vclk),
   
   .scandoubler_disable(forced_scandoubler),
   .hq2x(scale==1),
   .mono(0),
   .scanlines(0),
   .ypbpr_full(0),
   .line_start(0),

	.R(rr[7:2]),
	.G(gg[7:2]),
	.B(bb[7:2])
	
);

/////////////////  Tape In   /////////////////////////

wire tape_in;
assign tape_in = UART_RXD;

endmodule