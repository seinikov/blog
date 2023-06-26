# SDIO协议梳理附SD卡读写以及FATFS移植实例
SDIO为一种与SD-Card，SD-MMC或者SD总线设备通信的协议，基于命令和数据流。

## 物理层

| 协议项         | 协议内容                                                     |
| -------------- | ------------------------------------------------------------ |
| 通信模式       | 全双工                                                       |
| 连接线电气特性 | 6线通信：CLK（时钟线）,CMD（命令线）,DAT0~3（双向数据线）    |
|                | 3线供电：vdd（电源线），vss1（地线1），vss2（地线2）         |
| 电平值         | 默认模式，高速模式：3.3v                                     |
|                | SDR12,SDR25，SDR50，SDR104，DDR50,UHS156：1.8v               |
| 速度           | 默认模式，最高频率：25MHz，最高速度：12.5MB/s（11.92MB/s）4bit模式 |
|                | 高速模式：最高频率50MHz，最高速度：25MB/s（23.84MB/s）4bit模式 |
| 主从模式       | 默认速度：一主多从的同步星形拓扑结构，高速模式：一主一从的同步点对点结构 |

[![pCUi5lt.png](https://s1.ax1x.com/2023/06/26/pCUi5lt.png)](https://imgse.com/i/pCUi5lt)

SD卡在初始化时，处理命令会单独发送到每个卡，允许应用程序检测卡以及分配逻辑地址给物理卡槽，数据总是单独发送（接收）到每张卡，但为了简化卡的堆栈操作，在初始化过程结束后，所有命令都是同时发送所有卡，地址信息包含在命令包中。

SD总线允许数据线的动态配置。上电后，SD默认只使用DAT0来传输数据，即1bit-wide 模式。

> 所以在笔者使用6.8及以前版本CubeMx生成初始化代码时会有以下BUG：
>
> CubeMX会自动在初始化时将总线宽度设为 4bits，即以下式：
> 
> hsd.Init.BusWide = SDIO_BUS_WIDE_4B;
>
> 但SD手册规定初始只能是是1 bit，之后再由HAL_SD_ConfigWideBusOperation改变为4 bits,即必须为：
>
> hsd.Init.BusWide = SDIO_BUS_WIDE_1B;

初始化之后，主机可以改变总线宽度，即1bit-wide模式改为4bit-wide模式，这个功能允许硬件成本和系统性能之间的简单交换。当DAT1-DAT3没有使用的时候，主机的DAT先应该被设置为输入模式。

## 数据链路层

SD总线的通信是基于命令和数据流的。由一个起始位开始，由一个停止位终止，传输由高位到低位。

* 命令（Command）：命令就是一个标记，用于发起一个操纵。有主机发送到单个卡（寻址命令）或者所有卡（广播命令）。命令在CMD线上传输。
* 响应（Response）：相应是一个标记，从所寻址的卡或者所有卡（同步）发送给主机，作为接收到命令的应答。响应在CMD线上传输。
* 数据（DATA）：数据可以从主机到卡也可以从卡到主机。数据在DAT线上传输。

无响应操作与无数据操作：

[![pCN7sUS.png](https://s1.ax1x.com/2023/06/25/pCN7sUS.png)](https://imgse.com/i/pCN7sUS)

块读取操作：

[![pCN7cCQ.png](https://s1.ax1x.com/2023/06/25/pCN7cCQ.png)](https://imgse.com/i/pCN7cCQ)

块写入操作：

[![pCN7y4g.png](https://s1.ax1x.com/2023/06/25/pCN7y4g.png)](https://imgse.com/i/pCN7y4g)

### 命令帧格式：

| 项名称     | 位数  | 项内容                   |
| ---------- | ----- | ------------------------ |
| 起始位     | 1bit  | 常为‘0’                  |
| 传输来源位 | 1bit  | ‘1’=host command         |
| 内容位     | 38bit | 用于传输命令或命令参数   |
| CRC校验位  | 7bit  | 用于检查内容部分传输正确 |
| 停止位     | 1bit  | 常为‘1’                  |

命令帧一帧总长度为48bit。

[![pCN7b8J.png](https://s1.ax1x.com/2023/06/25/pCN7b8J.png)](https://imgse.com/i/pCN7b8J)

### 响应帧格式

响应帧有四种格式：

| 响应方式 | 格式                                                    |
| -------- | ------------------------------------------------------- |
| 1        | 方向位[1bit]+命令[8bit]+状态信息[32bit]+CRC[7bit]=48bit |
| 2        | 方向位+命令+CID/CSD寄存器+CRC=136bit                    |
| 3        | 方向位+命令+OCR寄存器+CRC=48bit                         |
| 4        | 方向位+命令+RCA寄存器+CRC=48bit                         |

[![pCN7LvR.png](https://s1.ax1x.com/2023/06/25/pCN7LvR.png)](https://imgse.com/i/pCN7LvR)

### 数据包格式

#### 常规8bit数据包

| 项名称         | 位数  | 项内容   |
| -------------- | ----- | -------- |
| 起始位         | 1bit  | ‘0’      |
| 数据位第一字节 | 8bit  | 数据内容 |
| 数据位第二字节 | 8bit  | 数据内容 |
| ***            | ***   | ***      |
| 数据位第n字节  | 8bit  | 数据内容 |
| CRC            |       | 校验     |
| 停止位         | 1bibt | ’1‘      |

下为数据包格式以及1b带宽模式和4b带宽模式下数据帧分配方法：

[![pCNHSUO.png](https://s1.ax1x.com/2023/06/25/pCNHSUO.png)](https://imgse.com/i/pCNHSUO)

#### 宽位数据包（巨帧数据包）

不按字节，传输512bit数据：

| 项名称 | 位数     | 项内容       |
| ------ | -------- | ------------ |
| 起始位 | 1bit     | 0            |
| 数据位 | 512bit？ | 不按字节传输 |
| CRC    |          | 校验         |
| 停止位 | 1bit     | 1            |

下为数据包格式以及1b带宽模式和4b带宽模式下数据帧分配方法：

[![pCNHP8H.png](https://s1.ax1x.com/2023/06/25/pCNHP8H.png)](https://imgse.com/i/pCNHP8H)

## 核心代码

都需要CubeMX配置硬件流控制，以此避免读写卡失败，事实上若不设置硬件流控制，单片可以可以读取到卡的信息，使用FATFS也可以在挂载和读写中返回成功值0，但实际上没有任何读写。

这里ST手册上的说明是：

>HW flow control
>
>The HW flow control functionality is used to avoid FIFO underrun (TX mode) and overrun (RX mode) errors.
>
>The behavior is to stop SDIO_CK and freeze SDIO state machines. The data transfer is stalled while the FIFO is unable to transmit or receive data. Only state machines clocked by SDIOCLK are frozen, the AHB interface is still alive. The FIFO can thus be filled or emptied even if flow control is activated.
>
>To enable HW flow control, the SDIO_CLKCR[14] register bit must be set to 1. After reset Flow Control is disabled.

### 无FATFS读写512字节数据

```c
#define SD_TIMEOUT             ((uint32_t)100000000)      /* 超时时间 */
#define SD_TRANSFER_OK         ((uint8_t)0x00)
#define SD_TRANSFER_BUSY       ((uint8_t)0x01) 
#define SD_TOTAL_SIZE_BYTE(__Handle__)  (((uint64_t)((__Handle__)->SdCard.LogBlockNbr)*((__Handle__)->SdCard.LogBlockSize))>>0)
#define SD_TOTAL_SIZE_KB(__Handle__)    (((uint64_t)((__Handle__)->SdCard.LogBlockNbr)*((__Handle__)->SdCard.LogBlockSize))>>10)
#define SD_TOTAL_SIZE_MB(__Handle__)    (((uint64_t)((__Handle__)->SdCard.LogBlockNbr)*((__Handle__)->SdCard.LogBlockSize))>>20)
#define SD_TOTAL_SIZE_GB(__Handle__)    (((uint64_t)((__Handle__)->SdCard.LogBlockNbr)*((__Handle__)>SdCard.LogBlockSize))>>30)

uint8_t sdcard_read(uint8_t *pbuf,uint32_t saddr,uint32_t cnt){
    uint8_t sta=HAL_OK;
    uint32_t timeout=SD_TIMEOUT;
    uint32_t lesector=saddr;
    __disable_irq();
    sta=HAL_SD_ReadBlocks(&hsd,(uint8_t *)pbuf,lesector,cnt,SD_TIMEOUT);

    while (((HAL_SD_GetCardState(&hsd)==HAL_SD_CARD_TRANSFER)?SD_TRANSFER_OK:SD_TRANSFER_BUSY)!=SD_TRANSFER_OK){
        if(timeout--==0){
            sta=SD_TRANSFER_BUSY;
        }
    }
    __enable_irq();
    return sta;
}

uint8_t sdcard_write(uint8_t *pbuf,uint32_t saddr,uint32_t cnt){
    uint8_t sta=HAL_OK;
    uint32_t timeout=SD_TIMEOUT;
    uint32_t lesector=saddr;
    __disable_irq();
    sta=HAL_SD_WriteBlocks(&hsd,(uint8_t *)pbuf,lesector,cnt,SD_TIMEOUT);
    while (((HAL_SD_GetCardState(&hsd)==HAL_SD_CARD_TRANSFER)?SD_TRANSFER_OK:SD_TRANSFER_BUSY)!=SD_TRANSFER_OK){
        if(timeout--==0){
            sta=SD_TRANSFER_BUSY;
        }
    }
    __enable_irq();
    return sta;
}

void sd_write_test(uint32_t secaddr,uint32_t seccnt){
    uint16_t i;
    uint8_t *buf;
    uint8_t sta=0;
    uint8_t bufarr[512];
    for(i=0;i<512;i++){
        bufarr[i]=i;
    }
    buf=bufarr;
    sta=sdcard_write(buf,secaddr,seccnt);
    if(sta==0){
        printf("WRITE\r\n");
        printf("SECTOR %d DATA:\r\n", secaddr);
        for (i = 0; i <512; i++){
            printf("%x ", buf[i]);
        }
        printf("\r\nDATA ENDED\r\n");
    }
    else{
        printf("err:%d\r\n", sta);
    }
    return ;
}

void sd_read_test(uint32_t secaddr,uint32_t seccnt){
    uint16_t i;
    uint8_t *buf;
    uint8_t sta=0;
    uint8_t bufarr[512];
    buf=bufarr;
    sta=sdcard_read(buf,secaddr,seccnt);
    if(sta==0){
        printf("READ\r\n");
        printf("SECTOR %d DATA:\r\n", secaddr);
        for (i = 0; i <512; i++){
            printf("%x ", buf[i]);  /* 打印secaddr开始的扇区数据 */
        }
        printf("\r\nDATA ENDED\r\n");
    }
    else{
        printf("err:%d\r\n", sta);
    }
}
```

### 有FATFS读写文件核心代码

SDIO配置

```c
uint8_t BSP_SD_Init(void)
{
  uint8_t sd_state = MSD_OK;
  /* Check if the SD card is plugged in the slot */
  if (BSP_SD_IsDetected() != SD_PRESENT)
  {
    return MSD_ERROR;
  }
  /* HAL SD initialization */
  sd_state = HAL_SD_Init(&hsd);
  /* Configure SD Bus width (4 bits mode selected) */
  if (sd_state == MSD_OK)
  {
    /* Enable wide operation */
    if (HAL_SD_ConfigWideBusOperation(&hsd, SDIO_BUS_WIDE_4B) != HAL_OK)
    {
      sd_state = MSD_ERROR;
    }
  }

  return sd_state;
}
void MX_SDIO_SD_Init(void)
{
  hsd.Instance = SDIO;
  hsd.Init.ClockEdge = SDIO_CLOCK_EDGE_RISING;
  hsd.Init.ClockBypass = SDIO_CLOCK_BYPASS_DISABLE;
  hsd.Init.ClockPowerSave = SDIO_CLOCK_POWER_SAVE_DISABLE;
  hsd.Init.BusWide = SDIO_BUS_WIDE_1B;
  hsd.Init.HardwareFlowControl = SDIO_HARDWARE_FLOW_CONTROL_ENABLE;
  hsd.Init.ClockDiv = 2;
  
  BSP_SD_Init();
}
```

使用FATFS库进行文件读写

```c
void fs_test(void){
    FRESULT ret;
    FATFS *fs_obj;
    FIL *fil_obj;

    uint8_t rd_buf[32]={0};
    uint16_t fsize=0;
    uint16_t rd_count,wd_count;
    
    printf("malloc before\r\n");
    fs_obj=mymalloc(0,sizeof(FATFS));
    fil_obj=mymalloc(0,sizeof(FIL));
    printf("malloc after,fs_obj:%p,fil_obj:%p\r\n",fs_obj,fil_obj);
    
    ret=f_mount(fs_obj,"0:",1);
    printf("after mount,ret=%d\r\n",ret);
    if(ret){
        printf("mount fail,error ret=%d\r\n",ret);
    }
    else{
        printf("mount success\r\n");
    }
    
    ret=f_open(fil_obj,"0:test.txt",FA_OPEN_ALWAYS|FA_READ|FA_WRITE);
    printf("after open,ret=%d\r\n",ret);
    if(ret){
        printf("open fail,error ret=%d\r\n",ret);
    }
    else{
        printf("open success\r\n");
    }
    
    ret=f_write(fil_obj,"FATFS-TEST",11,(UINT *)&wd_count);//结尾默认带EOF，所以需要将文件指针指向EOF前面
    printf("after write,ret=%d\r\n",ret);
    f_lseek(fil_obj,10);
    f_printf(fil_obj," SDCARD00\r\n");

    f_lseek(fil_obj,0);
    fsize=f_size(fil_obj);
    printf("fsize=%d\r\n",fsize);

    ret=f_read(fil_obj,(void *)rd_buf,(UINT)fsize,(UINT *)&rd_count);
    printf("after read,ret=%d,rd_count=%d,rd_buf size=%d\r\n",ret,rd_count,strlen((const char *)rd_buf));
    if(rd_count!=0){
        printf("rd_count:%d rd_buf:%s\r\n",rd_count,rd_buf);
    }
    
    f_close(fil_obj);
    myfree(0,fs_obj);
    myfree(0,fil_obj);
}
```

自定义堆区内存管理：

```c
#include "malloc.h"

#if !(__ARMCC_VERSION >= 6010050)   /* 不是AC6编译器，即使用AC5编译器时 */

/* 内存池(64字节对齐) */
static __align(64) uint8_t mem1base[MEM1_MAX_SIZE];                                     /* 内部SRAM内存池 */

/* 内存管理表 */
static MT_TYPE mem1mapbase[MEM1_ALLOC_TABLE_SIZE];                                                      /* 内部SRAM内存池MAP */

#else   /* 是AC6编译器，使用AC6编译器时 */

/* mem2base的地址：0x68000000，需要根据SRAM_BASE_ADDR的值计算，AC6不支持at宏定义表达式，因此只能手动计算
   如果SRAM_BASE_ADDR有改动，请自己重新计算，并修改 #if 和 __at_ 的: 0X68000000为新计算的值，否则会报错！
 */
#if SRAM_BASE_ADDR != 0x68000000
#error SRAM_BASE_ADDR changed! Please recalculate!
#endif

/* mem2mapbase的地址，如果地址有变更，这里会报错提示修改，修改方法同上 */
#if (SRAM_BASE_ADDR + MEM2_MAX_SIZE) != 0X680F0C00
#error SRAM_BASE_ADDR + MEM2_MAX_SIZE changed! Please recalculate!
#endif


/* 内存池(64字节对齐) */
static __ALIGNED(64) uint8_t mem1base[MEM1_MAX_SIZE];                                                       /* 内部SRAM内存池 */
static __ALIGNED(64) uint8_t mem2base[MEM2_MAX_SIZE] __attribute__((section(".bss.ARM.__at_0X68000000")));  /* 外扩SRAM内存池 */

/* 内存管理表 */
static MT_TYPE mem1mapbase[MEM1_ALLOC_TABLE_SIZE];                                                          /* 内部SRAM内存池MAP */
static MT_TYPE mem2mapbase[MEM2_ALLOC_TABLE_SIZE] __attribute__((section(".bss.ARM.__at_0X680F0C00")));     /* 外扩SRAM内存池MAP */

#endif

/* 内存管理参数 */
const uint32_t memtblsize[SRAMBANK] = {MEM1_ALLOC_TABLE_SIZE
                                       };       /* 内存表大小 */

const uint32_t memblksize[SRAMBANK] = {MEM1_BLOCK_SIZE
                                      };        /* 内存分块大小 */

const uint32_t memsize[SRAMBANK] = {MEM1_MAX_SIZE
                                   };           /* 内存总大小 */

/* 内存管理控制器 */
struct _m_mallco_dev mallco_dev =
{
    my_mem_init,                    /* 内存初始化 */
    my_mem_perused,                 /* 内存使用率 */
    mem1base,            /* 内存池 */
    mem1mapbase,       /* 内存管理状态表 */
    0,                    /* 内存管理未就绪 */
};

/**
 * @brief       复制内存
 * @param       *des : 目的地址
 * @param       *src : 源地址
 * @param       n    : 需要复制的内存长度(字节为单位)
 * @retval      无
 */
void my_mem_copy(void *des, void *src, uint32_t n)
{
    uint8_t *xdes = des;
    uint8_t *xsrc = src;

    while (n--)*xdes++ = *xsrc++;
}

/**
 * @brief       设置内存值
 * @param       *s    : 内存首地址
 * @param       c     : 要设置的值
 * @param       count : 需要设置的内存大小(字节为单位)
 * @retval      无
 */
void my_mem_set(void *s, uint8_t c, uint32_t count)
{
    uint8_t *xs = s;

    while (count--)*xs++ = c;
}

/**
 * @brief       内存管理初始化
 * @param       memx : 所属内存块
 * @retval      无
 */
void my_mem_init(uint8_t memx)
{
    uint8_t mttsize = sizeof(MT_TYPE);  /* 获取memmap数组的类型长度(uint16_t /uint32_t)*/
    my_mem_set(mallco_dev.memmap[memx], 0, memtblsize[memx]*mttsize); /* 内存状态表数据清零 */
    mallco_dev.memrdy[memx] = 1;        /* 内存管理初始化OK */
}

/**
 * @brief       获取内存使用率
 * @param       memx : 所属内存块
 * @retval      使用率(扩大了10倍,0~1000,代表0.0%~100.0%)
 */
uint16_t my_mem_perused(uint8_t memx)
{
    uint32_t used = 0;
    uint32_t i;

    for (i = 0; i < memtblsize[memx]; i++)
    {
        if (mallco_dev.memmap[memx][i])used++;
    }

    return (used * 1000) / (memtblsize[memx]);
}

/**
 * @brief       内存分配(内部调用)
 * @param       memx : 所属内存块
 * @param       size : 要分配的内存大小(字节)
 * @retval      内存偏移地址
 *   @arg       0 ~ 0XFFFFFFFE : 有效的内存偏移地址
 *   @arg       0XFFFFFFFF     : 无效的内存偏移地址
 */
static uint32_t my_mem_malloc(uint8_t memx, uint32_t size)
{
    signed long offset = 0;
    uint32_t nmemb;     /* 需要的内存块数 */
    uint32_t cmemb = 0; /* 连续空内存块数 */
    uint32_t i;

    if (!mallco_dev.memrdy[memx])
    {
        mallco_dev.init(memx);          /* 未初始化,先执行初始化 */
    }
    
    if (size == 0) return 0XFFFFFFFF;   /* 不需要分配 */

    nmemb = size / memblksize[memx];    /* 获取需要分配的连续内存块数 */

    if (size % memblksize[memx]) nmemb++;

    for (offset = memtblsize[memx] - 1; offset >= 0; offset--)  /* 搜索整个内存控制区 */
    {
        if (!mallco_dev.memmap[memx][offset])
        {
            cmemb++;            /* 连续空内存块数增加 */
        }
        else 
        {
            cmemb = 0;          /* 连续内存块清零 */
        }
        
        if (cmemb == nmemb)     /* 找到了连续nmemb个空内存块 */
        {
            for (i = 0; i < nmemb; i++) /* 标注内存块非空 */
            {
                mallco_dev.memmap[memx][offset + i] = nmemb;
            }

            return (offset * memblksize[memx]); /* 返回偏移地址 */
        }
    }

    return 0XFFFFFFFF;  /* 未找到符合分配条件的内存块 */
}

/**
 * @brief       释放内存(内部调用)
 * @param       memx   : 所属内存块
 * @param       offset : 内存地址偏移
 * @retval      释放结果
 *   @arg       0, 释放成功;
 *   @arg       1, 释放失败;
 *   @arg       2, 超区域了(失败);
 */
static uint8_t my_mem_free(uint8_t memx, uint32_t offset)
{
    int i;

    if (!mallco_dev.memrdy[memx])   /* 未初始化,先执行初始化 */
    {
        mallco_dev.init(memx);
        return 1;                   /* 未初始化 */
    }

    if (offset < memsize[memx])     /* 偏移在内存池内. */
    {
        int index = offset / memblksize[memx];      /* 偏移所在内存块号码 */
        int nmemb = mallco_dev.memmap[memx][index]; /* 内存块数量 */

        for (i = 0; i < nmemb; i++)                 /* 内存块清零 */
        {
            mallco_dev.memmap[memx][index + i] = 0;
        }

        return 0;
    }
    else
    {
        return 2;   /* 偏移超区了. */
    }
}

/**
 * @brief       释放内存(外部调用)
 * @param       memx : 所属内存块
 * @param       ptr  : 内存首地址
 * @retval      无
 */
void myfree(uint8_t memx, void *ptr)
{
    uint32_t offset;

    if (ptr == NULL)return;     /* 地址为0. */

    offset = (uint32_t)ptr - (uint32_t)mallco_dev.membase[memx];
    my_mem_free(memx, offset);  /* 释放内存 */
}

/**
 * @brief       分配内存(外部调用)
 * @param       memx : 所属内存块
 * @param       size : 要分配的内存大小(字节)
 * @retval      分配到的内存首地址.
 */
void *mymalloc(uint8_t memx, uint32_t size)
{
    uint32_t offset;
    offset = my_mem_malloc(memx, size);

    if (offset == 0XFFFFFFFF)   /* 申请出错 */
    {
        return NULL;            /* 返回空(0) */
    }
    else    /* 申请没问题, 返回首地址 */
    {
        return (void *)((uint32_t)mallco_dev.membase[memx] + offset);
    }
}

/**
 * @brief       重新分配内存(外部调用)
 * @param       memx : 所属内存块
 * @param       *ptr : 旧内存首地址
 * @param       size : 要分配的内存大小(字节)
 * @retval      新分配到的内存首地址.
 */
void *myrealloc(uint8_t memx, void *ptr, uint32_t size)
{
    uint32_t offset;
    offset = my_mem_malloc(memx, size);

    if (offset == 0XFFFFFFFF)   /* 申请出错 */
    {
        return NULL;            /* 返回空(0) */
    }
    else    /* 申请没问题, 返回首地址 */
    {
        my_mem_copy((void *)((uint32_t)mallco_dev.membase[memx] + offset), ptr, size); /* 拷贝旧内存内容到新内存 */
        myfree(memx, ptr);  /* 释放旧内存 */
        return (void *)((uint32_t)mallco_dev.membase[memx] + offset);   /* 返回新内存首地址 */
    }
}
```

