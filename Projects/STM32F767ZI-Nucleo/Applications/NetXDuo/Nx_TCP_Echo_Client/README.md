
##  <b>Nx_TCP_Echo_Client application description</b>

This application provides an example of Azure RTOS NetXDuo stack usage.
It shows how to develop a NetXDuo TCP client to communicate with a remote sever using the NetXDuo TCP socket API.

The main entry function tx_application_define() is then called by ThreadX during kernel start, at this stage, all NetXDuo resources are created.

 + A <i> NX_PACKET_POOL </i> is allocated
 + A <i>NX_IP</i> instance using that pool is initialized
 + The <i>ARP</i>, <i>ICMP</i>, <i>UDP</i> and <i>TCP</i> protocols are enabled for the <i>NX_IP</i> instance
 + A <i>DHCP</i> client is created.

The application then creates 2 threads with the same priorities:

 + **AppMainThread** (priority 10, PreemtionThreashold 10) : created with the <i>TX_AUTO_START</i> flag to start automatically.
 + **AppTCPThread** (priority 10, PreemtionThreashold 10) : created with the <i>TX_DONT_START</i> flag to be started later.

The **AppMainThread** starts and perform the following actions:

  + Starts the DHCP client
  + Waits for the IP address resolution
  + Resumes the **AppTCPThread**

The **AppTCPThread**, once started:

  + Creates a <i>TCP</i> socket
  + Connects to the remote <i>TCP</i> server on the predefined port
  + On connection success, the <i>TCP</i> client sends a <i>MAX_PACKET_COUNT</i> messages to the server.
  + At each message sent, the <i>TCP</i> client reads the sever response and prints it on the hyper-terminal and the green led is toggled.

####  <b>Expected success behavior</b>

 + The board IP address is printed on the HyperTerminal
 + The response messages sent by the server are printed on the HyerTerminal
 + If the [echotool](https://github.com/PavelBansky/EchoTool/releases/tag/v1.5.0.0) utility is used the message sent by the client are displayed on the PC console.
 + A summary message similar to the following is printed on the HyperTerminal and the green LED is toggling.
 ```
  SUCCESS : 10 / 10 packets sent
```

#### <b>Error behaviors</b>

+ The Red LED is toggling to indicate any error that have occurred.
+ In case the message exchange is not completed a failure message is printed on the HyperTerminal showing the actual sent message compared to the expected ones.

#### <b>Assumptions if any</b>

- The Application is using the DHCP to acquire IP address, thus a DHCP server should be reachable by the board in the LAN used to test the application.
- The application is configuring the Ethernet IP with a static predefined <i>MAC Address</i>, make sure to change it in case multiple boards are connected on the same LAN to avoid any potential network traffic issues.
- The _MAC Address_ is defined in the `main.c`

```
void MX_ETH_Init(void)
{

  /* USER CODE BEGIN ETH_Init 0 */

  /* USER CODE END ETH_Init 0 */

  /* USER CODE BEGIN ETH_Init 1 */

  /* USER CODE END ETH_Init 1 */
  heth.Instance = ETH;
  heth.Init.MACAddr[0] =   0x00;
  heth.Init.MACAddr[1] =   0x80;
  heth.Init.MACAddr[2] =   0xE1;
  heth.Init.MACAddr[3] =   0x00;
  heth.Init.MACAddr[4] =   0x00;
  heth.Init.MACAddr[5] =   0x00;
```
#### <b>Known limitations</b>
None

### <b>Notes</b>

 1.  If the user code size exceeds the DTCM-RAM size or starts from internal cacheable memories (SRAM1 and SRAM2), that is shared between several processors,
      then it is highly recommended to enable the CPU cache and maintain its coherence at application level.
      The address and the size of cacheable buffers (shared between CPU and other masters) must be properly updated to be aligned to cache line size (32 bytes).

#### <b>ThreadX usage hints</b>

 - ThreadX uses the Systick as time base, thus it is mandatory that the HAL uses a separate time base through the TIM IPs.
 - ThreadX is configured with 100 ticks/sec by default, this should be taken into account when using delays or timeouts at application. It is always possible to reconfigure it in the "tx_user.h", the "TX_TIMER_TICKS_PER_SECOND" define,but this should be reflected in "tx_initialize_low_level.s" file too.
 - ThreadX is disabling all interrupts during kernel start-up to avoid any unexpected behavior, therefore all system related calls (HAL, BSP) should be done either at the beginning of the application or inside the thread entry functions.
 - ThreadX offers the "tx_application_define()" function, that is automatically called by the tx_kernel_enter() API.
   It is highly recommended to use it to create all applications ThreadX related resources (threads, semaphores, memory pools...)  but it should not in any way contain a system API call (HAL or BSP).
 - Using dynamic memory allocation requires to apply some changes to the linker file.
   ThreadX needs to pass a pointer to the first free memory location in RAM to the tx_application_define() function,
   using the "first_unused_memory" argument.
   This require changes in the linker files to expose this memory location.
    + For EWARM add the following section into the .icf file:
     ```
	 place in RAM_region    { last section FREE_MEM };
	 ```
    + For MDK-ARM:
	```
    either define the RW_IRAM1 region in the ".sct" file
    or modify the line below in "tx_low_level_initilize.s to match the memory region being used
        LDR r1, =|Image$$RW_IRAM1$$ZI$$Limit|
	```
    + For STM32CubeIDE add the following section into the .ld file:
	```
    ._threadx_heap :
      {
         . = ALIGN(8);
         __RAM_segment_used_end__ = .;
         . = . + 64K;
         . = ALIGN(8);
       } >RAM AT> RAM
	```

       The simplest way to provide memory for ThreadX is to define a new section, see ._threadx_heap above.
       In the example above the ThreadX heap size is set to 64KBytes.
       The ._threadx_heap must be located between the .bss and the ._user_heap_stack sections in the linker script.
       Caution: Make sure that ThreadX does not need more than the provided heap memory (64KBytes in this example).
       Read more in STM32CubeIDE User Guide, chapter: "Linker script".

    + The "tx_initialize_low_level.s" should be also modified to enable the "USE_DYNAMIC_MEMORY_ALLOCATION" flag.

#### <b>NetX Duo usage hints</b>

 - The NetXDuo application needs to allocate the <b> <i> NX_PACKET </i> </b> pool in a dedicated section that is configured as below is an example
 of the section declaration for different IDEs.
 + For EWARM ".icf" file
   ```
   define symbol __ICFEDIT_region_NXDATA_start__  = 0x20060200;
   define symbol __ICFEDIT_region_NXDATA_end__   = 0x2007FFFF;
   define region NXAppPool_region  = mem:[from __ICFEDIT_region_NXDATA_start__ to __ICFEDIT_region_NXDATA_end__];
   place in NXAppPool_region { section .NXAppPoolSection};
   ```
   + For MDK-ARM
   ```
    RW_NXDriverSection 0x20060200 0x7800  {
  *(.NetXPoolSection)
  }
   ```
   + For STM32CubeIDE ".ld" file
   ```
   .nx_section 0x20060200 (NOLOAD): {
     *(.NXAppPoolSection)
     } >RAM
   ```

  this section is then used in the <code> app_azure_rtos.c</code> file to force the <code>nx_byte_pool_buffer</code> allocation.

```
/* USER CODE BEGIN NX_Pool_Buffer */

#if defined ( __ICCARM__ ) /* IAR Compiler */
#pragma location = ".NetXPoolSection"

#elif defined ( __CC_ARM ) /* MDK ARM Compiler */
__attribute__((section(".NetXPoolSection")))

#elif defined ( __GNUC__ ) /* GNU Compiler */
__attribute__((section(".NetXPoolSection")))
#endif

/* USER CODE END NX_Pool_Buffer */
__ALIGN_BEGIN static UCHAR nx_byte_pool_buffer[NX_APP_MEM_POOL_SIZE] __ALIGN_END;
static TX_BYTE_POOL nx_app_byte_pool;
```
For more details about the MPU configuration please refer to the [AN4838](https://www.st.com/resource/en/application_note/dm00272912-managing-memory-protection-unit-in-stm32-mcus-stmicroelectronics.pdf)


### <b>Keywords</b>

RTOS, Network, ThreadX, NetXDuo, TCP, UART

### <b>Hardware and Software environment</b>

  - This application runs on STM32F767ZIxx devices
  - This application has been tested with STMicroelectronics STM32F767ZI-NUCLEO board MB1137 Revision B-01
    and can be easily tailored to any other supported device and development board.

 - This application uses USART3 to display logs, the hyperterminal configuration is as follows:
      - BaudRate = 115200 baud
      - Word Length = 8 Bits
      - Stop Bit = 1
      - Parity = None
      - Flow control = None

###  <b>How to use it ?</b>

In order to make the program work, you must do the following :

 - Open your preferred toolchain
 - Edit the file <code> NetXDuo/App/app_netxduo.h</code> and correctly define the <TCP_SERVER_ADDRESS> and <TCP_SERVER_PORT> to connect on.
 - Run the [echotool](https://github.com/PavelBansky/EchoTool/releases/tag/v1.5.0.0) utility on a windows console as following:

       c:\> .\echotool.exe /p tcp /s <TCP_SERVER_PORT>

       Example : c:\> .\echotool.exe /p tcp /s 6001

 - Rebuild all files and load your image into target memory
 - Run the application
