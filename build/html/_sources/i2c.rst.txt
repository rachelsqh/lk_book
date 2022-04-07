i2c分析(待修正）
^^^^^^^^^^^^^^^^
i2c时序
"""""""
- 时序图：

img:i2c_time.png

  - 时序描述：I2C是一个能够支持多个设备的总线，包含一条双向串行数据线SDA，一条串行时钟线SCL。每个连接到总线的设备都有一个独立的地址，主机可以通过该地址来访问不同设备。主机通过SDA线发送设备地址（SLAVE_ADDRESS）查找从机，SLAVE_ADDRESS可以是7位或10位，紧跟着SLAVE_ADDRESS的一个数据位用来表示数据传输方向，即第8位或11位。为0时表示写数据，为1时表示读数据。SDA 线上的数据必须在时钟的高电平周期保持稳定，数据线的高或低电平状态只有在 SCL 线的时钟信号是低电平时才能改变。换言之，SCL为高电平时表示有效数据，SDA为高电平表示“1”，低电平表示“0”；SCL为低电平时表示无效数据，此时SDA会进行电平切换。

  - 起始条件S：当SCL高电平时，SDA由高电平向低电平转换；
  - 停止条件P：当SCL高电平时，SDA由低电平向高电平转换。
起始和停止条件一般由主机产生。总线在起始条件后处于busy的状态，在停止条件的某段时间后，总线才再次处于空闲状态。

  - 数据格式：传输的每个字节必须为8位，而总字节数不受限制。每个字节后必须跟一个响应位。首先开始传输的是数据最高位，即MSB位。如果此时从机正忙于其他功能，如正在中断服务程序，则需要使SCL线保持低电平迫使主机进入等待状态，直到从机准备完成。

  - 响应ACK时序图：
       i2c_ack.png
       描述：1个字节后，从设备如果正确接收，则在下一个时钟周期拉低SDA电平，表示ACK信号，如果从设备没反映，表示NOACK，每个字节后都要求ACK信号。
        每个字节后会跟随一个ACK信号。ACK bit使得接收者通知发送者已经成功接收数据并准备接收下一个数据。所有的时钟脉冲包括ACK信号对应的时钟脉冲都是由master产生的。

        ACK信号：发送者在ACK时钟脉冲期间释放SDA线，接收者可以将SDA拉低并在时钟信号为高时保持低电平。

        NACK信号：当在第9个时钟脉冲的时候SDA线保持高电平，就被定义为NACK信号。Master要么产生STOP条件来放弃这次传输，或者重复START条件来发起一个新的开始。
	
	所以，谁读谁回ACK/NACK。

gpio模拟i2c时序
"""""""""""""""
- start:先是把SDA和SCL电平都拉高，延时后将SDA电平拉低，使其出现下降沿，再延时后SCL电平也置低。
#define I2C_SDA_1()  GPIOB->BSRR = GPIO_Pin_7       // SDA线置高电平
#define I2C_SDA_0()  GPIOB->BRR = GPIO_Pin_7        // SDA线置低电平

#define I2C_SCL_1()  GPIOB->BSRR = GPIO_Pin_6       // SCL线置高电平
#define I2C_SCL_0()  GPIOB->BRR = GPIO_Pin_6        // SCL线置低电平

/**
  * @brief   产生起始条件
  * @param   无 
  * @retval  无
  */
void i2c_Start(void)
{
    I2C_SDA_1();
    I2C_SCL_1();
    Delay();
    I2C_SDA_0();
    Delay();
    I2C_SCL_0();
    Delay();
}

- stop:在SCL高电平时，SDA出现一个上升沿，即产生停止信号。

/**
  * @brief   产生停止条件
  * @param   无 
  * @retval  无
  */
void i2c_Stop(void)
{
    I2C_SDA_0();
    I2C_SCL_1();
    Delay();
    I2C_SDA_1();
}

- ACK/NACK:
/**
  * @brief   产生应答信号ACK
  * @param   无
  * @retval  无
  */
void i2c_Ack(void)
{
    I2C_SDA_0();
    Delay();
    I2C_SCL_1();
    Delay();
    I2C_SCL_0();
    Delay();
    I2C_SDA_1();	// CPU释放SDA
}


/**
  * @brief   产生非应答信号NACK
  * @param   无
  * @retval  无
  */
void i2c_NAck(void)
{
    I2C_SDA_1();
    Delay();
    I2C_SCL_1();
    Delay();
    I2C_SCL_0();
    Delay();
}

读取应答信号：
define I2C_SDA_READ()  (GPIOB->IDR & GPIO_Pin_7)     // 读取SDA电平状态

/**
  * @brief   CPU产生一个时钟，读取应答信号ACK
  * @param   无
  * @retval  返回0表示应答信号，返回1表示非应答信号
  */
uint8_t i2c_WaitAck(void)
{
    uint8_t re;

    I2C_SDA_1();	// CPU释放SDA总线
    Delay();
    I2C_SCL_1();	// CPU拉高SCL电平，发送一个时钟, 此时会返回ACK应答
    Delay();
    if (I2C_SDA_READ() != 0)	// 判断读取的SDA电平状态
    {
	re = 1;
    }
    else
    {
	re = 0;
    }
    I2C_SCL_0();
    Delay();
    
    return re;
}



- 发送数据：I2C协议中指出，发送的每个字节必须为8位，并且从最高位开始传输。我们只要通过SDA将该字节一位一位发送出去即可。
/**
  * @brief   发送一个字节的数据
  * @param   
  *   @arg   byte：等待发送的字节
  * @retval  无
  */
void i2c_SendByte(uint8_t byte)
{
    uint8_t i;

    /* 先发送字节的最高位 */
    for (i=0; i<8; i++)
    {		
        if (byte & 0x80)    // 判断最高位为1或0
	{
	    I2C_SDA_1();
	}
	else
	{
	    I2C_SDA_0();
	}

	Delay();
	I2C_SCL_1();
	Delay();	
	I2C_SCL_0();
	if(i == 7)
	{
	    I2C_SDA_1();    // 释放总线
	}
	byte <<= 1;	    // 左移1位，准备发送下一位
	Delay();
    }
}

数据发送：
/**
  * @brief   在一定的延时时间内等待应答
  * @param
  *   @arg delay：延时时间 
  * @retval  无
  */
void i2c_WaitAck_Delay(delay)
{
    uint8_t ack_status;

    while(delay--)
    {
        ack_status = i2c_WaitAck();
        if(ack_status)         // 如果是非应答，主机需要产生停止信号终止传输
        {
            i2c_Stop();
	    return 0;
        }
        else
        {
            break;
        }
    }
}

/**
  * @brief   写一个字节到EEPROM
  * @param  
  *   @arg WriteAddr：要写入的内存地址 
  *   @arg pBuffer：缓冲区指针
  * @retval  无
  */
uint32_t I2C_ByteWrite(u8 WriteAddr, u8 pBuffer)
{			
    i2c_Start();

    i2c_SendByte(0xA0);	       // 发送设备地址
    i2c_WaitAck_Delay(1000);
		
    i2c_SendByte(WriteAddr);   // 发送要写入的内存地址
    i2c_WaitAck_Delay(1000);

    i2c_SendByte(pBuffer);     // 开始写入数据
    i2c_WaitAck_Delay(1000);

    i2c_Stop();
    return 1;
}

- 接收数据：读取字节数据也是逐位完成的，读到的第一位是最高位。


/**
  * @brief   接收一个字节的数据
  * @param   无
  * @retval  接收到的数据
  */
uint8_t i2c_ReadByte(void)
{
    uint8_t i;
    uint8_t value = 0;

    for (i=0; i<8; i++)
    {
	value <<= 1;
	I2C_SCL_1();
	Delay();
	if(I2C_SDA_READ())
	{
	    value++;
	}
	I2C_SCL_0();
	Delay();
    }

    return value;
}
从EEPROM读取数据，只是多了个起始条件，需要先将设备地址和数据内存地址写入，然后才开始读取。

/**
  * @brief   读取字节数据到EEPROM
  * @param  
  *   @arg WriteAddr：要写入的内存地址 
  *   @arg pBuffer：缓冲区指针
  *   @arg numByteToRead：读取的字节数
  * @retval  无
  */
uint32_t I2C_Read(u8 WriteAddr, u8* pBuffer, u8 numByteToRead)
{
    /* 第一次起始条件 */
    i2c_Start();

    i2c_SendByte(0xA0);	       // 此处是写指令
    i2c_WaitAck_Delay(1000);

    i2c_SendByte(WriteAddr);
    i2c_WaitAck_Delay(1000);

    /* 第二次起始条件 */
    i2c_Start();

    i2c_SendByte(0xA1);	       // 此处是读指令
    i2c_WaitAck_Delay(1000);	

    while(numByteToRead)
    {
	*pBuffer = i2c_ReadByte();
		
	/* 每读完1个字节后，需要发送Ack，最后一个字节发Nack */
	if (numByteToRead != 1)
	{
	    i2c_Ack();
	}
	else
	{
	    i2c_NAck();/* 需要精确的解析 */
	}
        
        pBuffer++;
        numByteToRead--;
    }

    i2c_Stop();
    return 0;
}


i2c驱动分析
"""""""""""
我们以触摸屏驱动为例进行说明。

- i2c控制器驱动：eg20t:

- tsc2007 注册
- 故障总结：

总结
"""""""
- i2c信号：注意ACK/NACK信号；
- i2c匹配依据：i2c 从设备地址；
- i2c速率总结：半双工
- 驱动架构总结：
  - 主设备驱动注册：
  - 从设备注册：
  - 匹配结构：
  



















