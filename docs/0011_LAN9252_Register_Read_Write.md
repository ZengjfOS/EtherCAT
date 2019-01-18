# LAN9252 Register Read Write

## 地址访问说明

目前的开发板既支持4bit间接地址访问，也支持内存地址直接访问，目前STM32代码使用的内存直接访问内的方式。参考：[《STM32F407 EtherCAT Project Base》](0003_STM32F407_EtherCAT_Project_Base.md#寻址方式)；

## CSR（Controller Slave Register）读写

```C
void PDI_Isr(void)
{
    if(bEscIntEnabled)
    {
        /* get the AL event register */
        UINT16  ALEvent = HW_GetALEventRegister_Isr();              // 读取事件寄存器
        ALEvent = SWAPWORD(ALEvent);

        [...省略]
    }

    [...省略]
}

UINT16 HW_GetALEventRegister_Isr(void)
{
    ISR_GetInterruptRegister();                                     // 获取事件寄存器
    return EscALEvent.Word;
}

static void ISR_GetInterruptRegister(void)
{
    HW_EscReadIsr((MEM_ADDR *)&EscALEvent.Word, 0x220, 2);          // 读取寄存器
}

 void HW_EscReadIsr( MEM_ADDR *pData, UINT16 Address, UINT16 Len )
{

    UINT16 i;
    UINT8 *pTmpData = (UINT8 *)pData;

    /* send the address and command to the ESC */

    /* loop for all bytes to be read */
    while ( Len > 0 )
    {

        if (Address >= 0x1000)                              // 貌似LAN9252不会大于这个
        {
            i = Len;
        }
        else                                                // 矫正长度，主要是处理字节访问对齐的问题
        {
            i= (Len > 4) ? 4 : Len;

            if(Address & 01) 
            {
               i=1;
            }
            else if (Address & 02)
            {
               i= (i&1) ? 1:2;
            }
            else if (i == 03)
            {
                i=1;
            }
        }
 
        PMPReadDRegister(pTmpData, Address,i);              // 读取寄存器

        Len -= i;
        pTmpData += i;
        Address += i;
    }
   
}

void PMPReadDRegister(UINT8 *ReadBuffer, UINT16 Address, UINT16 Count)
{
    if (Address >= 0x1000)                                  // The Process RAM starts at address 1000h.
    {
        PMPReadPDRamRegister(ReadBuffer, Address,Count);
    }
    else
    {
        PMPReadRegUsingCSR(ReadBuffer, Address,Count);      // 读取控制寄存器
    }
}

void PMPReadRegUsingCSR(UINT8 *ReadBuffer, UINT16 Address, UINT8 Count)
{
    UINT32_VAL param32_1 = {0};
    UINT8 i = 0;
    UINT16_VAL wAddr;
    wAddr.Val = Address;

    param32_1.v[0] = wAddr.byte.LB;
    param32_1.v[1] = wAddr.byte.HB;
    param32_1.v[2] = Count;
    param32_1.v[3] = ESC_READ_BYTE;

    PMPWriteDWord (CSR_CMD_REG, param32_1.Val);             // 写寄存器

    do
    {
        param32_1.Val = PMPReadDWord (ESC_CSR_CMD_REG);     // 读寄存器
    }while(param32_1.v[3] & ESC_CSR_BUSY);


    param32_1.Val = PMPReadDWord (ESC_CSR_DATA_REG);

    
    for(i=0;i<Count;i++)
        ReadBuffer[i] = param32_1.v[i];
   
    return;
}

void PMPWriteDWord (UINT16 Address, UINT32 Val)
{
   
    UINT32_VAL res;
    res.Val=Val;
    
    /* Transfer data to the memory */
    
    // *(uint32_t *) (Bank1_SRAM4_ADDR + Address) = Val;
    *(uint16_t *) (Bank1_SRAM4_ADDR + Address) = res.w[0];              // 直接地址访问

    /* Increment the address*/
    Address += 2;
    *(uint16_t *) (Bank1_SRAM4_ADDR + Address) = res.w[1];              // 直接地址访问

  
}

UINT32 PMPReadDWord (UINT16 Address)
{
    
    UINT32_VAL res;
      
    //res.Val = (UINT32)(*(volatile UINT32 *)(Bank1_SRAM4_ADDR+(Address)));
    res.w[0]= *(__IO uint16_t*) (Bank1_SRAM4_ADDR + Address);           // 直接地址访问
    /* Increment the address*/
    Address += 2;
    res.w[1]= *(__IO uint16_t*) (Bank1_SRAM4_ADDR + Address);           // 直接地址访问
    

    return res.Val;
}
```
