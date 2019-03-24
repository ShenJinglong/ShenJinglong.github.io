---
title: 8b/10b编解码器的Verilog实现
date: 2019-02-08 13:28:13
categories:
- Verilog
tags:
- Verilog
- FPGA
copyright: true
---

我用于测试的图片大小为 269*269 的，如果你的图片大小不同，就还得修改一些地方。写的比较仓促，我水平也有限，将就参考参考吧

<!-- more -->

###### 这是我用于测试的图片：

![1.jpg](http://i350.photobucket.com/albums/q439/wood_/face_zpsqpii7rp3.jpg)

### 8b/10b 编码器

首先生成需要编码的文件，这个文件以 0xBC 为同步码，以 0xFD 为帧头，以 0xFB 为帧尾，帧头和帧尾之间包含 64 字节的数据，数据包于数据包之间的同步码为 8 个，这个地方我用 C++ 和 OpenCV 实现，代码如下

```cpp
// get_data.cpp: 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include <opencv2/opencv.hpp>
#include <fstream>

using namespace cv;
using namespace std;

void same_step(ofstream &ofs);
void begin_data(ofstream &ofs);
void end_data(ofstream &ofs);
bool send_data(ofstream &ofs, Mat_<uchar>::iterator &it_begin, Mat_<uchar>::iterator &it_end);

int main()
{
	Mat img = imread("1.jpg");
	if (!img.data)
	{
		cout << "当前文件夹下没有命名为 1.jpg 的图片存在" << endl;
		return -1;
	}
	cvtColor(img, img, COLOR_BGR2GRAY);
	ofstream out;
	out.open("./data.txt");
	
	Mat_<uchar>::iterator it_begin = img.begin<uchar>();
	Mat_<uchar>::iterator it_end = img.end<uchar>();

	same_step(out);
	while (1)
	{
		if (it_begin != it_end)
		{
			begin_data(out);
			if (send_data(out, it_begin, it_end))
			{
				end_data(out);
				same_step(out);
			}
			else
			{
				end_data(out);
				same_step(out);
				break;
			}
		}
		else
		{
			break;
		}
	}
	

	imshow("img", img);
	waitKey(0);
    return 0;
}

void same_step(ofstream &ofs)
{
	for (int i = 0; i < 8; ++i)
		ofs << int(0xBC) << endl;
}

void begin_data(ofstream &ofs)
{
	ofs << int(0xFD) << endl;
}

void end_data(ofstream &ofs)
{
	ofs << int(0xFB) << endl;
}

bool send_data(ofstream &ofs, Mat_<uchar>::iterator &it_begin, Mat_<uchar>::iterator &it_end)
{
	for (int i = 0; i < 64; ++i)
	{
		if (it_begin != it_end)
		{
			ofs << int(*it_begin) << endl;
			++it_begin;
		}
		else
		{
			ofs << int(0xFB) << endl;
		}
	}
	if (it_begin != it_end)
		return true;
	else
		return false;
}
```

运行这个程序就可以生成名为 data.txt 的待编码文件，下面用 Verilog 实现编码器

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 2018/12/13 16:17:52
// Design Name: 
// Module Name: top
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////


module top(
    input [7:0] data_i,
    input clk,
    output [9:0] data_o
    );
    reg [9:0] trans_data = {2'b01, 8'hbc};
    
    reg [15:0] clk_cnt = 0;
    reg clk2MHz = 0;
    always @(posedge clk)
        begin
            if (clk_cnt != 250)
                clk_cnt <= clk_cnt + 1;
            else
                begin
                    clk2MHz <= ~clk2MHz;
                    clk_cnt <= 0;
                end
        end
    
    always @(data_i)
        begin
            clk2MHz <= 0;
            clk_cnt <= 0;
        end
    
     
    reg [7:0] same_step_cnt = 0;
    reg [7:0] data_cnt = 0;
    
    parameter INIT = 3'b000;
    parameter STATE_JUDGE = 3'b101;
    
    reg [2:0] cur_state = INIT;
   
    reg [7:0] data_buf;
    
    reg trans_flag = 0;
    always @(posedge clk2MHz)
        begin
            case (cur_state)
                INIT:
                    begin
                        data_buf <= data_i;
                        cur_state <= STATE_JUDGE;
                        trans_data <= {2'b01, 8'hbc};
                    end
                STATE_JUDGE:
                    begin
                        if (data_buf == 8'hbc && data_i == 8'hbc && data_cnt == 0)
                            begin
                                trans_data <= {2'b01, 8'hbc};
                                data_buf <= data_i;
                            end
                        else if (data_buf == 8'hbc && data_i == 8'hfd && data_cnt == 0)
                            begin
                                trans_data <= {2'b01, 8'hfd};
                                data_buf <= data_i;
                            end
                        else if (data_buf == 8'hfb && data_i == 8'hbc && data_cnt == 0)
                            begin
                                trans_data <= {2'b01, 8'hbc};
                                data_buf <= data_i;
                            end
                        else
                            begin
                                if (data_cnt != 64)
                                    begin
                                        trans_data <= {2'b00, data_i};
                                        data_cnt <= data_cnt + 1;
                                        data_buf <= data_i;
                                    end
                                else
                                    begin
                                        trans_data <= {2'b01, 8'hfb};
                                        data_cnt <= 0;
                                        data_buf <= data_i;
                                    end    
                            end
                    end
            endcase
            trans_flag <= ~trans_flag;
        end
    
    data_encoder encoder_(
        .clk(clk),
        .i_data(trans_data),
        .o_data(data_o),
        .trans_triger(trans_flag)
        );
endmodule

module data_encoder(
	input clk,
	input [9:0] i_data,
	input trans_triger,
	output reg [9:0] o_data
	);
	
	parameter DATA526 = 4'b0000;
	parameter DATA324 = 4'b0001;
	parameter JUDGE526 = 4'b0010;
	parameter JUDGE324 = 4'b0011;
	
	parameter JUDGECODE = 4'b0100;
	
	parameter K8210 = 4'b0101;
	parameter KJUDGE = 4'b0110;
	
	reg [3:0] cur_state = JUDGECODE, next_state = JUDGECODE;
	
	reg rd_flag = 0;
	reg trans_flag = 1;
	
	always @(trans_triger)
		begin
			trans_flag <= 1;
		end
	
	always @(posedge clk)
		begin
			if (trans_flag)
				cur_state <= next_state;
		end
	
	always @(cur_state)
		begin
			case (cur_state)
			    JUDGECODE:
                    begin
                        if (i_data[9:8] == 2'b00)
                            next_state <= DATA526;
                        else if (i_data[9:8] == 2'b01)
                            next_state <= K8210;
                        else
                            begin
                                trans_flag <= 0;
                                next_state <= JUDGECODE;
                            end
                    end     
				DATA526:
					begin
						o_data[9:4] <= encoder526(i_data[4:0]);
						next_state <= JUDGE526;
					end
				DATA324:
					begin
						o_data[3:0] <= encoder324(i_data[7:5]);
						next_state <= JUDGE324;
					end
				JUDGE526:
					begin
						if (o_data[9] + o_data[8] + o_data[7] + o_data[6] + o_data[5] + o_data[4] != 3)
							rd_flag <= ~rd_flag;
						next_state <= DATA324;
					end
				JUDGE324:
					begin
						if (o_data[3] + o_data[2] + o_data[1] + o_data[0] != 2)
							rd_flag <= ~rd_flag;
						next_state <= JUDGECODE;
						trans_flag <= 0;
					end
                K8210:
                    begin
                        o_data <= kencoder(i_data);
                        next_state <= KJUDGE;
                    end
                KJUDGE:
                    begin
                        if (o_data[0] + o_data[1] + o_data[2] + o_data[3] + o_data[4] + o_data[5] + o_data[6] + o_data[7] + o_data[8] + o_data[9] != 5)
                            rd_flag <= ~rd_flag;
                        next_state <= JUDGECODE;
                        trans_flag <= 0;
                    end
			endcase
		end
	
	function [5:0] encoder526;
		input [4:0] i_data526;
		if (rd_flag == 0)
			case (i_data526)
				5'b00000: encoder526 = 6'b100111;
				5'b00001: encoder526 = 6'b011101;
				5'b00010: encoder526 = 6'b101101;
				5'b00011: encoder526 = 6'b110001;
				5'b00100: encoder526 = 6'b110101;
				5'b00101: encoder526 = 6'b101001;
				5'b00110: encoder526 = 6'b011001;
				5'b00111: encoder526 = 6'b111000;
				5'b01000: encoder526 = 6'b111001;
				5'b01001: encoder526 = 6'b100101;
				5'b01010: encoder526 = 6'b010101;
				5'b01011: encoder526 = 6'b110100;
				5'b01100: encoder526 = 6'b001101;
				5'b01101: encoder526 = 6'b101100;
				5'b01110: encoder526 = 6'b011100;
				5'b01111: encoder526 = 6'b010111;
				5'b10000: encoder526 = 6'b011011;
				5'b10001: encoder526 = 6'b100011;
				5'b10010: encoder526 = 6'b010011;
				5'b10011: encoder526 = 6'b110010;
				5'b10100: encoder526 = 6'b001011;
				5'b10101: encoder526 = 6'b101010;
				5'b10110: encoder526 = 6'b011010;
				5'b10111: encoder526 = 6'b111010;
				5'b11000: encoder526 = 6'b110011;
				5'b11001: encoder526 = 6'b100110;
				5'b11010: encoder526 = 6'b010110;
				5'b11011: encoder526 = 6'b110110;
				5'b11100: encoder526 = 6'b001110;
				5'b11101: encoder526 = 6'b101110;
				5'b11110: encoder526 = 6'b011110;
				5'b11111: encoder526 = 6'b101011;
			endcase
		else
			case (i_data526)
				5'b00000: encoder526 = 6'b011000;
				5'b00001: encoder526 = 6'b100010;
				5'b00010: encoder526 = 6'b010010;
				5'b00011: encoder526 = 6'b110001;
				5'b00100: encoder526 = 6'b001010;
				5'b00101: encoder526 = 6'b101001;
				5'b00110: encoder526 = 6'b011001;
				5'b00111: encoder526 = 6'b000111;
				5'b01000: encoder526 = 6'b000110;
				5'b01001: encoder526 = 6'b100101;
				5'b01010: encoder526 = 6'b010101;
				5'b01011: encoder526 = 6'b110100;
				5'b01100: encoder526 = 6'b001101;
				5'b01101: encoder526 = 6'b101100;
				5'b01110: encoder526 = 6'b011100;
				5'b01111: encoder526 = 6'b101000;
				5'b10000: encoder526 = 6'b100100;
				5'b10001: encoder526 = 6'b100011;
				5'b10010: encoder526 = 6'b010011;
				5'b10011: encoder526 = 6'b110010;
				5'b10100: encoder526 = 6'b001011;
				5'b10101: encoder526 = 6'b101010;
				5'b10110: encoder526 = 6'b011010;
				5'b10111: encoder526 = 6'b000101;
				5'b11000: encoder526 = 6'b001100;
				5'b11001: encoder526 = 6'b100110;
				5'b11010: encoder526 = 6'b010110;
				5'b11011: encoder526 = 6'b001001;
				5'b11100: encoder526 = 6'b001110;
				5'b11101: encoder526 = 6'b010001;
				5'b11110: encoder526 = 6'b100001;
				5'b11111: encoder526 = 6'b010100;
			endcase
	endfunction
	
	function [3:0] encoder324;
		input [2:0] i_data324;
		if (rd_flag == 0)	
			case (i_data324)
				3'b000: encoder324 = 4'b1011;
				3'b001: encoder324 = 4'b1001;
				3'b010: encoder324 = 4'b0101;
				3'b011: encoder324 = 4'b1100;
				3'b100: encoder324 = 4'b1101;
				3'b101: encoder324 = 4'b1010;
				3'b110: encoder324 = 4'b0110;
				3'b111:
					begin
						if (i_data[4:0] == 5'd17 || i_data[4:0] == 5'd18 || i_data[4:0] == 5'd20)
							encoder324 = 4'b0111;
						else
							encoder324 = 4'b1110;
					end
			endcase
		else
			case (i_data324)
				3'b000: encoder324 = 4'b0100;
				3'b001: encoder324 = 4'b1001;
				3'b010: encoder324 = 4'b0101;
				3'b011: encoder324 = 4'b0011;
				3'b100: encoder324 = 4'b0010;
				3'b101: encoder324 = 4'b1010;
				3'b110: encoder324 = 4'b0110;
				3'b111:
					begin
						if (i_data[4:0] == 5'd11 || i_data[4:0] == 5'd13 || i_data[4:0] == 5'd14)
							encoder324 = 4'b1000;
						else
							encoder324 = 4'b0001;
					end
			endcase
	endfunction
	
	function [9:0] kencoder;
	   input [7:0] i_data;
	   if (rd_flag == 0)
	       begin
	           case (i_data)
	               8'b00011100: kencoder = 10'b0011110100;
	               8'b00111100: kencoder = 10'b0011111001;
	               8'b01011100: kencoder = 10'b0011110101;
	               8'b01111100: kencoder = 10'b0011110011;
	               8'b10011100: kencoder = 10'b0011110010;
	               8'b10111100: kencoder = 10'b0011111010;
	               8'b11011100: kencoder = 10'b0011110110;
	               8'b11111100: kencoder = 10'b0011111000;
	               8'b11110111: kencoder = 10'b1110101000;
	               8'b11111011: kencoder = 10'b1101101000;
	               8'b11111101: kencoder = 10'b1011101000;
	               8'b11111110: kencoder = 10'b0111101000;
	           endcase
	       end
	   else
	       begin
	           case (i_data)
	               8'b00011100: kencoder = 10'b1100001011;
	               8'b00111100: kencoder = 10'b1100000110;
	               8'b01011100: kencoder = 10'b1100001010;
	               8'b01111100: kencoder = 10'b1100001100;
	               8'b10011100: kencoder = 10'b1100001101;
	               8'b10111100: kencoder = 10'b1100000101;
	               8'b11011100: kencoder = 10'b1100001001;
	               8'b11111100: kencoder = 10'b1100000111;
	               8'b11110111: kencoder = 10'b0001010111;
	               8'b11111011: kencoder = 10'b0010010111;
	               8'b11111101: kencoder = 10'b0100010111;
	               8'b11111110: kencoder = 10'b1000010111;	               
	           endcase
	       end
	endfunction
endmodule
```

然后编写 testbench 进行仿真

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 2018/12/13 17:54:13
// Design Name: 
// Module Name: tb
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////


module tb(

    );
    reg [7:0] i_data = 8'hbc;
    wire [9:0] o_data;
    reg clk = 0;
    always clk = #0.5 ~clk;
    reg [31:0] cnt = 0;
    
    integer fp_r;
    integer fp_w;
    integer count;
    
    initial
        begin
            count = 0;
            fp_r = $fopen("D:/Mine/FPGA/data.txt", "r"); //这一行的地址为你刚刚生成的 data.txt 的地址
            fp_w = $fopen("D:/Mine/FPGA/after_code.txt", "w"); //这一行的地址为编码后文件 after_code.txt 的输出地址
        end
    
    always @(posedge clk)
        begin
            if (cnt != 500)
                cnt <= cnt + 1;
            else
                begin
                    $fscanf(fp_r, "%d", i_data);
                    if (count != 0)
                        $fwrite(fp_w, "%d\n", o_data);
                    cnt <= 0;
                    if (count != 83702) //这个地方的数字改为你生成的 data.txt 文件里数据的个数
                        count <= count + 1;
                    else
                        begin
                            $fclose(fp_r);
                            $fclose(fp_w);
                        end
                end
        end
    
    top test(
        .clk(clk),
        .data_i(i_data),
        .data_o(o_data)
        );
endmodule
```

仿真完成之后即可生成名为 after_code.txt 的编码之后的文件
哦，对了！要运行这个仿真的话需要改一些地方，要改的地方我写注释里了

### 8b/10b 解码器

首先用 Verilog 实现解码器

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 2018/12/14 20:41:55
// Design Name: 
// Module Name: decoder
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////


module decoder(
    input clk,
    input [9:0] data_i,
    output reg [7:0] data_o
    );
        
    always @(posedge clk)
        begin
            case (data_i)
                10'b0011110100, 10'b1100001011: data_o <= 8'b00011100;
                10'b0011111001, 10'b1100000110: data_o <= 8'b00111100;
                10'b0011110101, 10'b1100001010: data_o <= 8'b01011100;
                10'b0011110011, 10'b1100001100: data_o <= 8'b01111100;
                10'b0011110010, 10'b1100001101: data_o <= 8'b10011100;
                10'b0011111010, 10'b1100000101: data_o <= 8'b10111100;
                10'b0011110110, 10'b1100001001: data_o <= 8'b11011100;
                10'b0011111000, 10'b1100000111: data_o <= 8'b11111100;
                10'b1110101000, 10'b0001010111: data_o <= 8'b11110111;
                10'b1101101000, 10'b0010010111: data_o <= 8'b11111011;
                10'b1011101000, 10'b0100010111: data_o <= 8'b11111101;
                10'b0111101000, 10'b1000010111: data_o <= 8'b11111110;
                default:
                    begin
                        data_o[4:0] <= func_625(data_i[9:4]);
                        data_o[7:5] <= func_423(data_i[3:0]);
                    end
            endcase
        end
        
    function [4:0] func_625;
        input [5:0] data_in;
        case (data_in)
            6'b100111, 6'b011000: func_625 = 5'b00000;
            6'b011101, 6'b100010: func_625 = 5'b00001;
            6'b101101, 6'b010010: func_625 = 5'b00010;
            6'b110001: func_625 = 5'b00011;
            6'b110101, 6'b001010: func_625 = 5'b00100;
            6'b101001: func_625 = 5'b00101;
            6'b011001: func_625 = 5'b00110;
            6'b111000, 6'b000111: func_625 = 5'b00111;
            6'b111001, 6'b000110: func_625 = 5'b01000;
            6'b100101: func_625 = 5'b01001;
            6'b010101: func_625 = 5'b01010;
            6'b110100: func_625 = 5'b01011;
            6'b001101: func_625 = 5'b01100;
            6'b101100: func_625 = 5'b01101;
            6'b011100: func_625 = 5'b01110;
            6'b010111, 6'b101000: func_625 = 5'b01111;
            6'b011011, 6'b100100: func_625 = 5'b10000;
            6'b100011: func_625 = 5'b10001;
            6'b010011: func_625 = 5'b10010;
            6'b110010: func_625 = 5'b10011;
            6'b001011: func_625 = 5'b10100;
            6'b101010: func_625 = 5'b10101;
            6'b011010: func_625 = 5'b10110;
            6'b111010, 6'b000101: func_625 = 5'b10111;
            6'b110011, 6'b001100: func_625 = 5'b11000;
            6'b100110: func_625 = 5'b11001;
            6'b010110: func_625 = 5'b11010;
            6'b110110, 6'b001001: func_625 = 5'b11011;
            6'b001110: func_625 = 5'b11100;
            6'b101110, 6'b010001: func_625 = 5'b11101;
            6'b011110, 6'b100001: func_625 = 5'b11110;
            6'b101011, 6'b010100: func_625 = 5'b11111;
        endcase
    endfunction
    
    function [2:0] func_423;
        input [3:0] data_in;
        case (data_in)
            4'b1011, 4'b0100: func_423 = 3'b000;
            4'b1001: func_423 = 3'b001;
            4'b0101: func_423 = 3'b010;
            4'b1100, 4'b0011: func_423 = 3'b011;
            4'b1101, 4'b0010: func_423 = 3'b100;
            4'b1010: func_423 = 3'b101;
            4'b0110: func_423 = 3'b110;
            4'b1110, 4'b0001: func_423 = 3'b111;
            4'b0111, 4'b1000: func_423 = 3'b111;
        endcase
    endfunction
endmodule
```

然后编写 testbench 进行仿真

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 2018/12/14 20:42:10
// Design Name: 
// Module Name: tb
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////


module tb(

    );      
    reg [9:0] data_i = 10'b0;
    wire [7:0] data_o;
    reg clk = 0;
    always clk = #0.5 ~clk;
    reg [31:0] cnt = 0;
    
    integer fp_r;
    integer fp_w;
    integer count;
    
    reg clk2MHz = 0;
    
    initial
        begin
            fp_r = $fopen("D:/Mine/FPGA/after_code.txt", "r");//这一行的地址为你刚刚生成的 after_code.txt 的地址
            fp_w = $fopen("D:/Mine/FPGA/after_decode.txt", "w");//这一行的地址为解码后文件 after_decode.txt 的输出地址
            count = 0;
        end

    always @(posedge clk)
        begin
            if (cnt != 500)
                begin
                    cnt <= cnt + 1;
                    if (count != 0)
                        begin
                            if (cnt == 250)
                                clk2MHz <= ~clk2MHz;
                        end
                end
            else
                begin
                    $fscanf(fp_r, "%d", data_i);
                    cnt <= 0;
                    if (count != 0)
                        $fwrite(fp_w, "%d\n", data_o);
                        clk2MHz <= ~clk2MHz;
                    if (count != 83702)//这个地方的数字改为你生成的 after_code.txt 文件里数据的个数
                        count <= count + 1;
                    else
                        begin
                            $fclose(fp_r);
                            $fclose(fp_w);
                        end
                end
        end
    
    decoder test(
        .clk(clk2MHz),
        .data_i(data_i),
        .data_o(data_o)
        );
endmodule
```

emmmmm... 这个仿真同样也有需要改动的地方，就在注释那
仿真完成后即可得名为 after_decode.txt 的解码之后的文件，这个时候可以将 after_decode.txt 于 data.txt 进行比对，这两个文件的内容应该是一致的，然后就从 after_decode.txt 中恢复图片，这个地方仍用 C++ 和 OpenCV 实现，代码如下

```cpp
// recover.cpp: 定义控制台应用程序的入口点。
//

#include "stdafx.h"
#include <opencv2/opencv.hpp>
#include <fstream>
#include <iostream>

using namespace std;
using namespace cv;

uchar getdata(fstream &);

int main()
{
	fstream f("after_decode.txt", ios::in);
	int data;
	int data_pre;
	int rows, cols;
	cout << "这个地方不要输错了，不然图片恢复不了" << endl;
	cout << "输入图片的高度：";
	cin >> rows;
	cout << "输入图片的宽度：";
	cin >> cols;
	Mat img(rows, cols, CV_8UC1);
	
	for (int i = 0; i < rows; ++i)
	{
		for (int j = 0; j < cols; ++j)
		{
			img.at<uchar>(i, j) = getdata(f);
		}
	}

	imshow("img", img);
	imwrite("result.jpg", img);
	waitKey(0);
	return 0;
}

uchar getdata(fstream &f)
{
	static int data_cnt = 0;
	int val;
	if (data_cnt == 0)
	{
		while (1)
		{
			f >> val;
			if (val == 253)
				break;
		}
	}
	f >> val;
	++data_cnt;
	if (data_cnt == 64)
		data_cnt = 0;
	return uchar(val);
}
```

运行之后就可以看到你之前的图片了

