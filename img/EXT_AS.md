[TOC]
# Merlin Project
**Author ： YaphetSZz, X.L.Liu**  

| 日期      | 版本       | 描述                      | 作者         | Email  |
| ------    | ------    | ------                    | ------       | ------ |
| 22/2/28  | V1.0      | EXT设计文档初步整理                 |  **周莆钧**  | pujun_zhou@163.com |
| 22/3/14  | V1.1      | EXT设计文档整理                 |  **刘小龙、刘强**  | 2590116272@qq.com |

>参考资料  
[张师兄JAP论文](https://github.com/black20441/Image_bed/blob/71b4f81341a2b7cdefeaf8155d75ca40a84cef4e/EXT/A%20versatile%20neuromorphic%20system%20based%20on%20simpl.pdf)

## 架构概述

### 背景

V1.0 
2022/2/28
这是BICS两年前张晨明师兄的设计，初期未整理文档，如今，在芯片测试前夕，BICS-IC-2组对EXT芯片的AS文档做初步整理。
EXT芯片设计有9个计算核，在二维欧几里得空间中排列为3*3的正方形，每个核心的东南西北四个方向都有路由器模块，用以跨核心的数据交互

V1.1
2020/3/14
根据

### 文件结构说明

- rtl文件根目录“IC_2/EXT_Project/rtl”
  - fpgahead.v **项目的头文件，定义了一些参数**
  - internal_sram20x256.v **SRAM模块**
  - IO_PAD.v **后端设计添加的IO控制模块**
  - memory.v **memory设计**
  - Neuromorphic.v **芯片设计顶层**
  - Neuromorphic_controller.v
  - Neuromorphic_pad.v **后端设计顶层**
  - reset_syn.v **异步复位模块**
  - router.v **router模块**
  - spi_controller.v **spi控制模块**
  - sram_controller.v **sram控制模块**
  - sync_fifo.v **异步fifo模块**
  - "./core0_0"和"./core" (这两个里面放的是core的设计，core0_0在工程中例化了1次，core例化了9次，共组成10个core的阵列，二者核心功能相同，不同的是core0_0有些信号被拉到了顶层neuromorphic_controller中用以进行顶层控制)
    - controller.v **core的控制模块**
    - core.v **core的顶层设计模块**
    - neuron.v **神经元模块，主要负责神经元计算**
    - scheduler.v **core内的存储器模块，用来存储输入spike**
  
### 特性和功能概述

- 通用类脑计算芯片设计，内部集成10个计算核
- 每个计算核包含256个神经元和256*256个突触
- 每个计算核的东南西北各有4个路由模块用来进行核间数据传递
- 所有核心的timestep相同
- 顶层通过spi协议与上位机进行通信

## 功能说明

### 结构框图

#### 整体系统架构

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/系统架构.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">系统架构</div>
</center>

#### 核心通信架构

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/核心通信架构.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">核心通信架构</div>
</center>

#### 普通核心通信

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/核心通信.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">普通核心通信</div>
</center>

#### 核心0通信

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/核心0通信.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">核心0通信</div>
</center>

### 架构说明

功能模块描述：

- **Neuromorphic_controller模块**：神经形态芯片顶层控制模块，其内部包含控制逻辑，主要功能如下：
  - 接收外部控制信号，输出脉冲读使能、神经脉冲id与神经形态数据
  - 接收spi控制模块的写使能、写数据，输出spi读请求、读数据
  - 接收sram控制模块的读数据，输出sram读请求、读地址、写请求、写地址、写数据
  - 向core memory模块输出写请求、写数据
  - 接收core0_0 scheduler模块的num_scheduler，向所有核输出scheduler写请求、delivery_time、activity_vector数据
  - 与所有的计算核的controller模块进行数据交互（向controller模块传输核开始信号、全局时间、激活开关使能，接收核心状态、神经元状态、神经元索引、激活开关请求、核心0_0的脉冲读使能、脉冲神经元索引、及所有核的脉冲读数据和膜电位）
- **reset_syn模块**：异步复位模块，控制芯片异步复位
- **spi_controller模块**：spi控制模块，内含spi控制逻辑，主要功能如下：
  - 控制spi接口与外部设备进行读、写数据操作
  - 控制与神经形态controller模块进行读、写数据操作
- **sram_controller模块**：sram控制模块，其内部包含片内sram控制逻辑，主要功能如下：
  - 控制片内sram的read与write
  - 控制与片外sram的数据传输
- **core：计算核**
  - **Controller模块**
    - 接收Neuromorphic_controller模块的控制命令
    - 向Neuromorphic_controller模块传输spikes_read_data数据
    - 向neuron模块输出控制命令，接收neuron的start信号与索引
    - 向scheduler模块输出scheduler读请求、time_step
  - **neuron模块**
    - 读取片内sram memory数据
    - 向片内sram写入膜电位数据
    - 向core controller模块的进行数据传输
    - 接收scheduler模块输出的活动矢量数据
    - 接收router模块的local数据包
    - 阈值模式、泄漏模式和重置模式可配置。
  - **scheduler模块**
    - 接收神经形态controller模块的写请求、delivery_time、activity_vector
    - 接收router模块的local数据包
    - 接收core controller模块的读请求、time_step
    - 将scheduler_read_data数据传给neuron模块的activity_vector端口
  - **memory模块**
    - 接收来自神经形态controller模块的写请求、写数据
    - 接收neuron模块的读、写请求、地址、数据
  - **router模块**
    - 接收north、south、west、east、local五个异步FIFO的的数据
    -向north、south、west、east、local五个异步FIFO输出数据

## 性能指标

## 工作数据流

### 工作模式

- 工作模式1
配置文件写入，通过改变外部控制命令的写入，从spi接口向片外sram写入配置文件，向芯片内部写入三个位置控制数据，向scheduler写入activity_vector数据
- 工作模式2
神经形态控制，通过从外部控制命令的写入核心开始命令，初始化每一个运算核的memory，并等待更新memory数据
- 工作模式3
启动网络计算，

### 数据流

#### 工作模式1数据流

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/烧录文件.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">烧录文件</div>
</center>
工作流程如下：

1. 外部控制命令从spi接口写入等待与初始化指令
2. 外部控制命令改为写命令，从spi接口写入三个位置数据
3. 外部控制命令改为写SRAM命令，从spi接口向片外SRAM写入含有神经网络权重数据的数据包
4. 外部控制命令改为写ACTIVITY命令，从spi接口向scheduler写入activity_vector数据

外部控制命令如下：

1. control signal为NULL=4'd0时，将核开始信号复位
2. control signal为INIT=4'd1时，将核开始信号复位
3. control signal为WRITE_COMMAND=4'd2时，从spi接口将数据写入core_location、activity_location、read_location三个存储位置的寄存器中
4. control signal为WRITE_SRAM=4'd3时，从spi接口向外部SRAM写入数据
5. control signal为WRITE_ACTIVITY=4'd4时，从spi接口得到activity_vector数据，确定核的调度器索引后得到调度器写请求，向核的调度器写入activity_vector数据
6. control signal为CORES_START=4'd5时，如果神经形态状态为WAIT_UPDATE_TRIGGER，将core_location数据传给核心开始信号
7. control signal为READ_CHIP=4'd6时，从spi接口读取控制信号、核状态、调度器、核存储器索引等数据

#### 工作模式2数据流

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/神经形态控制.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">神经形态控制</div>
</center>
工作流程如下：

1. 在配置文件写入完成后，外部控制命令改为CORES_START=4'd5，此时神经形态控制程序开始运行，状态跳转至初始化memory
2. 初始化memory状态，向每一个核心的memory模块写入来自外部SRAM的的权重数据，等待向所有核心的memory写入完成，状态跳转至等待更新触发状态
3. 等待更新触发状态，等待来自核0的神经元状态数据，当神经元的状态数据为初始化memory（4'd2）时，此时跳转至更新memory状态
4. 更新memory状态，向所有核的memory模块写入新的来自外部SRAM的的权重数据，并向核控制器传递核更新完成标志，状态跳转至等待更新触发状态

神经形态状态控制命令如下：

1. neuromorphic_state为NULL=4'd0时，等待外部控制信号给CORES_START=4'd5信号后开始INIT_MEMORY状态
2. neuromorphic_state为INIT_MEMORY=4'd1时，向每一个核的memory写入从外部sram读取的配置数据
3. neuromorphic_state为WAIT_UPDATE_TRIGGER=4'd2时，等待从核0传来的神经元状态数据，若neuron_state=4'd2，此时状态跳转至UPDATE_MEMORY，否则在本状态等待
4. neuromorphic_state为UPDATE_MEMORY=4'd3时，重新向每一个核的memory写入从外部sram读取的配置数据

#### 工作模式3数据流

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.jsdelivr.net/gh/fhjlkl/pic_bed/img/核心0运算.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">核心运算</div>
</center>

工作流程如下：

1. 核心控制模块接收来自神经形态控制模块的核心开始命令，核心状态开始跳转至初始化ACTIVITY状态
2. 初始化ACTIVITY状态，向scheduler发送读请求，核心状态跳转至初始化CORE
3. 初始化CORE状态，向神经元模块发送开始指令后，状态跳转至切换神经元状态
4. 切换神经元状态，等待神经元运算完成，并返回神经元运算完成信号，直到所有神经元都完成计算，则跳转至切换ACTIVITY状态
5. 切换ACTIVITY状态，发送activity切换请求，状态跳转至切换ACTIVITY状态，并根据activity切换使能信号判断该核跳转至初始化CORE状态还是切换到NULL状态

神经元运算状态跳转流程如下：

1. NULL=4'd0，对神经元脉冲、神经元结束、本地发射标志和memory写请求清零，等待神经元开始信号到来，跳转至INIT_MEMORY状态，并发送memory读请求
2. INIT_MEMORY=4'd1，等待一个memory延时，然后跳转至INIT_MEMORY状态
3. INIT_NEURON = 4'd2，将从memory读取的664位数据写入神经元模块内的寄存器中等待运算，然后跳转至INIT_AXON状态
4. INIT_AXON = 4'd3，获取轴突的类型数据，然后跳转至SYNAPTIC状态，直到遍历完所有的轴突，跳转至LEAK状态
5. SYNAPTIC = 4'd4，根据获取的轴突类型获取神经元权重数据，然后将膜电位进行神经元加权运算，运算完成后状态跳转至INIT_AXON
6. LEAK = 4'd5，根据从memory读取配置文件的第568位泄露反转标志，对膜电位进行泄露电流加权运算，运算完成后状态跳转至FIRE_RESET
7. FIRE_RESET = 4'd6，启动膜电位复位状态，根据复位模式对膜电位进行复位，并将local发射标志置一，在local发射数据包中写入传递时间+router的x、y坐标+axon数据，然后状态跳转至WRITEBACK
8. WRITEBACK = 4'd7，发送神经元运算结束标志，与memory写请求，向memory写入新的膜电位数据，local发射标志复位，状态跳转至NULL。

## 时钟、复位、初始化

### 时钟域划分

### 复位信号描述

### 初始化确定

## 地址划分

### SRAM的地址划分

## 寄存器说明

### Neuromorphic模块寄存器表

| 寄存器名称    | 位宽     | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| control_signal_lat |   [3:0]   |  | 控制信号锁存 |

### Neuromorphic_controller模块寄存器表

| 寄存器名称    | 位宽     | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| spi_data          | [1023:0] |  | spi接口的数据 |
| num_command_byte  | [7:0]    |  | SPI写命令位数 |
| num_sram_byte     | [7:0]    |  | SPI向SRAM写入的位数 |
| num_activity_byte | [7:0]    |  | SPI向调度器写入的位数 |
| core_location     | [63:0]   |  | core位置有效数据 |
| activity_location | [63:0]   |  | 调度器位置有效数据 |
| read_location     | [63:0]   |  | 读取core位置有效数据 |
| scheduler_index   | [7:0]    |  | 调度器索引 |
| delivery_time_pre |  [4:0]   |  | 预设传输时间 |
| pre_time_step     |  [4:0]   |  | 预设时间步 |
| control_signal_latch   |  [3:0]   |  | 外部控制命令锁存器 |
| neuromorphic_state     |  [3:0]   |  | 神经形态状态 |
| core_index        |  [7:0]   |  | 核心索引 |
| core_valid_idx    |  [7:0]   |  | 核心有效索引 |
| count_clk         |  [3:0]   |  | 时钟计数 |
| num_sram_words    |  [7:0]   |  | SRAM队列写入字数 |
| sram_data_sequence     |  [703:0]   |  | SRAM数据队列 |
| memory_index        |  [7:0]   |  | 存储器索引 |

### 异步fifo寄存器表

| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| head    |  [3:0]   |  | 数据开头 |
| tail    |  [3:0]   |  | 数据结束 |
| count   |  [4:0]   |  | 队列计数 |
| data    |  [20:0]  |  | 存储数据 |
| fifo_read_data    |  [20:0]   |  | 读FIFO数据 |

### SPI控制器寄存器表

| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| mosi_data     | [7:0]  |  | spi输入数据 |
| num_mosi_bit  | [7:0]  |  | spi输入位数 |
| miso_data     | [1023:0]  |  | spi输出数据 |
| num_miso_bit  | [9:0]  |  | spi输出位数 |

### SRAM控制器寄存器表

| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| sram_state   | [3:0]  |  | sram状态 |
| count_clk    | [3:0]  |  | 时钟计数 |

### 路由模块寄存器表

packet_process Register
| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| x_dimension   | [3:0]  |  | x轴坐标 |
| y_dimension   | [7:4]  |  | y轴坐标 |
| axon_index    | [15:8] |  | 轴突索引 |
| delivery_time | [20:16] |  | 传递时间 |

### Core控制器寄存器表

| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| core_state   | [3:0]  |  | 核状态 |
| rst_idx      | [8:0]  |  | 复位索引 |
| sum_state    | [1:0]  |  | 脉冲计数状态 |
| sum_spikes   | [7:0]  |  | 脉冲计数 |
| spikes_neuron_idx   | [7:0]  |  | 脉冲神经元索引 |

### 神经元寄存器表

internal_read_data Register  
| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| membrane_potential   | [19:0]  |  | 膜电位数据 |
| axon_type            | [531:20]  |  | 256个轴突类型数据 |
| neuron_weight_vector[0]     | [540:532]|  | 神经元权重数据 |
| neuron_weight_vector[1]     | [549:541]|  | 神经元权重数据 |
| neuron_weight_vector[2]     | [558:550]|  | 神经元权重数据 |
| neuron_weight_vector[3]     | [567:559]|  | 神经元权重数据 |
| leak_reversal_flag          | [568]    |  | 泄露反转标志|
| pos_threshold               | [588:569]|  | 正阈值电压|
| neg_threshold               | [608:589]|  | 负阈值电压|
| reset_voltage               | [628:609]|  | 重置电压|
| neg_reset_mode              | [629]    |  | 负重置模式|
| reset_mode                  | [631:630]|  | 重置模式选择|
| packet_process              | [652:632]|  | router数据包 |
| leak_weight                 | [663:653]|  | 泄露电压权重数据|

### 调度器寄存器表

| 寄存器名称   | 位宽      | 地址      | 描述      |
| ------      | ------    | ------   | ------    |
| pre_delivery_time   | [4:0]  |  | 预设传输时间 |
| num_scheduler       | [5:0]  |  | 调度次数，写调度加一，读调度减一 |
| scheduler_delay     | [3:0]  |  | 调度器延时 |
| scheduler_Q         | [255:0]  |  | 调度器输出 |
| scheduler_BWEN      | [255:0]  |  | 调度器位写使能 |
| scheduler_A         | [4:0]  |  | 调度器地址输入 |
| scheduler_D         | [255:0]  |  | 调度器数据输入 |

## 功能点

### RISC-V模块

## 接口信号

## 验证关注点

### 整体验证关注点

### 模块验证关注点

## 资源评估

## 设计难点

## 文档说明