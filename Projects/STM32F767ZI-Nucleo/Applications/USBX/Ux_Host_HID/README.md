
## <b>Ux_Host_HID application description</b>

This application provides an example of Azure RTOS USBX stack usage .
It shows how to develop USB Host Human Interface "HID" able to enumerate and communicates with a mouse or a keyboard.

The application is designed to behave as an USB HID Host, the code provides required requests to properly enumerate
HID devices , HID Class APIs to decode HID reports received from a mouse or a keyboard and display data on uart HyperTerminal.

The main entry function tx_application_define() is then called by ThreadX during kernel start, at this stage, all USBx resources
are initialized, the HID Class driver and HID clients are registered.
The application creates 3 threads with different priorities :

  - usbx_app_thread_entry     (Priority : 25; Preemption threshold : 25) used to initialize USB OTG HAL HCD driver and start the Host.
  - hid_mouse_thread_entry    (Priority : 30; Preemption threshold : 30) used to decode HID reports received  from a mouse.
  - hid_keyboard_thread_entry (Priority : 30; Preemption threshold : 30) used to decode HID reports received  from a keyboard.

#### <b>Expected success behavior</b>

When a hid device is plugged to STM32F767ZI-Nucleo board, a Message will be displayed on the uart HyperTerminal showing
the Vendor ID and Product ID of the attached device.
After enumeration phase, a message will indicates that the device is ready for use.
The host must be able to properly decode HID reports sent by the corresponding device and display those information on the HyperTerminal.

The received HID reports are used by host to identify:
in case of a mouse
   - (x,y) mouse position
   - Wheel position
   - Pressed mouse buttons

in case of a keyboard
 - Pressed key

#### <b>Error behaviors</b>

Errors are detected such as (Unsupported device, Enumeration Fail) and the corresponding message is displayed on the HyperTerminal.

The Red LED is toggling to indicate any error that has occurred.

#### <b>Assumptions if any</b>

User is familiar with USB 2.0 "Universal Serial BUS" Specification and HID class Specification.

#### <b>Known limitations</b>

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
       } >RAM_D1 AT> RAM_D1
	```

       The simplest way to provide memory for ThreadX is to define a new section, see ._threadx_heap above.
       In the example above the ThreadX heap size is set to 64KBytes.
       The ._threadx_heap must be located between the .bss and the ._user_heap_stack sections in the linker script.
       Caution: Make sure that ThreadX does not need more than the provided heap memory (64KBytes in this example).
       Read more in STM32CubeIDE User Guide, chapter: "Linker script".

    + The "tx_initialize_low_level.s" should be also modified to enable the "USE_DYNAMIC_MEMORY_ALLOCATION" flag.

#### <b>USBX usage hints</b>

- The DTCM non cacheable memory (0x20000000-0x2001FFFF) is accessible by the USB-DMA.
- When using a cacheable memory (SRAM1 or SRAM2), make sure to configure the USB memory pool region attribute "Non-Cacheable" to ensure coherency between the CPU and the USB DMA.

### <b>Keywords</b>

Connectivity, USBX Host, ThreadX, USB, HID, Mouse, Keyboard, UART, USART,


### <b>Hardware and Software environment</b>

  - This application runs on STM32F767xx devices
  - This application has been tested with STMicroelectronics STM32F767ZI-NUCLEO board MB1137 Revision B-01
    and can be easily tailored to any other supported device and development board.

  - NUCLEO-F767ZI Set-up
    - Plug the USB HID device into the NUCLEO-F767ZI board through 'USB micro A-Male  to A-Female' cable to the connector:
      - CN13 : to use USB Full Speed OTG IP.
    - Connect ST-Link cable to the PC USB port to display data on the HyperTerminal.

    A virtual COM port will then appear in the HyperTerminal:
     - Hyperterminal configuration
       - Data Length = 8 Bits
       - One Stop Bit
       - No parity
       - BaudRate = 115200 baud
       - Flow control: None


### <b>How to use it ?</b>

In order to make the program work, you must do the following :

 - Open your preferred toolchain
 - Rebuild all files and load your image into target memory
 - Run the application

