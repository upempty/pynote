

- FreeRTOS task switch: vTaskStartScheduler to save the context and restore the new task
```
== https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/c9e3949f02f0350986f7a7df273e8bf2e9311d04/examples/cmake_example/main.c#L48

void main( void )
{
    static StaticTask_t exampleTaskTCB;
    static StackType_t exampleTaskStack[ configMINIMAL_STACK_SIZE ];

    ( void ) printf( "Example FreeRTOS Project\n" );

    ( void ) xTaskCreateStatic( exampleTask,
                                "example",
                                configMINIMAL_STACK_SIZE,
                                NULL,
                                configMAX_PRIORITIES - 1U,
                                &( exampleTaskStack[ 0 ] ),
                                &( exampleTaskTCB ) );

    /* Start the scheduler. */
    vTaskStartScheduler();

    for( ; ; )
    {
        /* Should not reach here. */
    }
}



void vTaskStartScheduler( void )
{
    BaseType_t xReturn;

    traceENTER_vTaskStartScheduler();
    ....


        /* The return value for xPortStartScheduler is not required
         * hence using a void datatype. */
        ( void ) xPortStartScheduler();



BaseType_t xPortStartScheduler( void )
{
    extern void( vPortStartFirstTask )( void );
    extern uint32_t _stack[];

    /* Setup the hardware to generate the tick.  Interrupts are disabled when
     * this function is called.
     *
     * This port uses an application defined callback function to install the tick
     * interrupt handler because the kernel will run on lots of different
     * MicroBlaze and FPGA configurations - not all of which will have the same
     * timer peripherals defined or available.  An example definition of
     * vApplicationSetupTimerInterrupt() is provided in the official demo
     * application that accompanies this port. */
    vApplicationSetupTimerInterrupt();

    /* Reuse the stack from main() as the stack for the interrupts/exceptions. */
    pulISRStack = ( uint32_t * ) _stack;

    /* Ensure there is enough space for the functions called from the interrupt
     * service routines to write back into the stack frame of the caller. */
    pulISRStack -= 2;

    /* Restore the context of the first task that is going to run.  From here
     * on, the created tasks will be executing. */
    vPortStartFirstTask();


/*
 * Called by portYIELD() or taskYIELD() to manually force a context switch.
 *
 * When a context switch is performed from the task level the saved task
 * context is made to look as if it occurred from within the tick ISR.  This
 * way the same restore context function can be used when restoring the context
 * saved from the ISR or that saved from a call to vPortYieldProcessor.
 */
void vPortYieldProcessor( void )
{
    /* Within an IRQ ISR the link register has an offset from the true return
     * address, but an SWI ISR does not.  Add the offset manually so the same
     * ISR return code can be used in both cases. */
    __asm volatile ( "ADD       LR, LR, #4" );

    /* Perform the context switch.  First save the context of the current task. */
    portSAVE_CONTEXT();

    /* Find the highest priority task that is ready to run. */
    vTaskSwitchContext();

    /* Restore the context of the new task. */
    portRESTORE_CONTEXT();
}




 */
#if( ( configMEMMODEL == portSMALL ) || ( configMEMMODEL == portMEDIUM ) )

    #define portSAVE_CONTEXT()                                          \
            {   __asm(" POPW  A ");                                     \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" PUSHW (RW0,RW1,RW2,RW3,RW4,RW5,RW6,RW7) ");     \
                __asm(" MOVW A, _pxCurrentTCB ");                       \
                __asm(" MOVW A, SP ");                                  \
                __asm(" SWAPW ");                                       \
                __asm(" MOVW @AL, AH ");                                \
                __asm(" OR   CCR,#H'20 ");                              \
            }

/*
 * Macro to restore a task context from the task stack. This is
 * effectively the reverse of SAVE_CONTEXT(). First the stack pointer
 * value (USP for SMALL and MEDIUM memory model amd USB:USP for COMPACT
 * and LARGE memory model ) is loaded from the task  control block. Next
 * the value of all the general purpose registers RW0-RW7 is retrieved.
 * Finally it copies of the context ( AH:AL, DPR:ADB, DTB:PCB, PC and PS)
 * of the task to be executed upon RETI from user stack to system stack.
 */

    #define portRESTORE_CONTEXT()                                       \
            {   __asm(" MOVW A, _pxCurrentTCB ");                       \
                __asm(" MOVW A, @A ");                                  \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" MOVW SP, A ");                                  \
                __asm(" POPW (RW0,RW1,RW2,RW3,RW4,RW5,RW6,RW7) ");      \
                __asm(" POPW  A ");                                     \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" PUSHW  A ");                                    \
                __asm(" AND  CCR,#H'DF ");                              \
                __asm(" POPW  A ");                                     \
                __asm(" OR   CCR,#H'20 ");                              \
                __asm(" PUSHW  A ");                                    \
            }


https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/c9e3949f02f0350986f7a7df273e8bf2e9311d04/portable/Softune/MB96340/port.c#L150

or suspend schduling inside:



 void vTaskSuspend( TaskHandle_t xTaskToSuspend )
     vTaskSwitchContext();
 
void vTaskSwitchContext( BaseType_t xCoreID )
   
                /* Select a new task to run. */
                taskSELECT_HIGHEST_PRIORITY_TASK( xCoreID );


 #define taskSELECT_HIGHEST_PRIORITY_TASK( xCoreID )    prvSelectHighestPriorityTask( xCoreID )

static void prvSelectHighestPriorityTask( BaseType_t xCoreID )
    prvYieldCore( x );
        portYIELD_CORE( xCoreID );  
        pxCurrentTCBs[ ( xCoreID ) ]->xTaskRunState = taskTASK_SCHEDULED_TO_YIELD; 
            #define portYIELD_CORE( x )    portYIELD()



/*
 * Manual context switch called by portYIELD or taskYIELD.
 *
 * The first thing we do is save the registers so we can use a naked attribute.
 */
void vPortYield( void ) __attribute__( ( naked ) );
void vPortYield( void )
{
    /* We want the stack of the task being saved to look exactly as if the task
     * was saved during a pre-emptive RTOS tick ISR.  Before calling an ISR the
     * msp430 places the status register onto the stack.  As this is a function
     * call and not an ISR we have to do this manually. */
    asm volatile ( "push    r2" );
    _DINT();

    /* Save the context of the current task. */
    portSAVE_CONTEXT();

    /* Switch to the highest priority task that is ready to run. */
    vTaskSwitchContext();

    /* Restore the context of the new task. */
    portRESTORE_CONTEXT();
}

```

- FreeRTOS is schdudled from task1->task2, and maybe ISR comes to interrupt current task, handle it and resume to specific task1.

  
```

timer sw的定义去跟ticks比较：

每个isr处理后去increase the ticks by one shot task: xTaskIncrementTick().
so that xTickcounter++. So this can be type of clock timer.



ISR task:

https://www.freertos.org/zh-cn-cmn-s/implementation/a00011.html
中断如果是tickisr这个周期时间到了触发这个TickISR的处理（硬件响应），这个timer的callback最后处理好后执行Return from ISR，这个control task是freertos app创建的。
里边可以有对这个timer到的时间到了以后的处理。这样不会阻塞中断上半部的时间。
如下也是类似的例子。
https://github.com/LetsControltheController/freertos-task-ISR/blob/c7eed0248de01dc98a077812a9d6391414361963/main/main.c#L26

#include <stdio.h>
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"


#define CONFIG_LED_PIN 2
#define ESP_INTR_FLAG_DEFAULT 0
#define CONFIG_BUTTON_PIN 0


TaskHandle_t ISR = NULL;

// interrupt service routine, called when the button is pressed
void IRAM_ATTR button_isr_handler(void* arg) {
  
 xTaskResumeFromISR(ISR);----- TCB_t * const pxTCB = xTaskToResume(ISR also); prvAddTaskToReadyList( pxTCB );--------------ISR resumed task which was pushed into task list to be ready to schedule. 
//portYIELD_FROM_ISR(  );
}

// task that will react to button clicks
 void button_task(void *arg)
{
bool led_status = false;
 while(1){  
vTaskSuspend(NULL);
      led_status = !led_status;
      gpio_set_level(CONFIG_LED_PIN, led_status);
printf("Button pressed!!!\n");
 }
}///////////////////////////////////////////////////////////////////


void app_main()
{
   
    gpio_pad_select_gpio(CONFIG_BUTTON_PIN);
  gpio_pad_select_gpio(CONFIG_LED_PIN);
  
  // set the correct direction
  gpio_set_direction(CONFIG_BUTTON_PIN, GPIO_MODE_INPUT);
    gpio_set_direction(CONFIG_LED_PIN, GPIO_MODE_OUTPUT);
  
  // enable interrupt on falling (1->0) edge for button pin
  gpio_set_intr_type(CONFIG_BUTTON_PIN, GPIO_INTR_NEGEDGE);

  
  //Install the driver’s GPIO ISR handler service, which allows per-pin GPIO interrupt handlers.
  // install ISR service with default configuration
  gpio_install_isr_service(ESP_INTR_FLAG_DEFAULT);
  
  // attach the interrupt service routine
  gpio_isr_handler_add(CONFIG_BUTTON_PIN, button_isr_handler, NULL);-----------------------------inside callback will resume to the below task as this call: xTaskResumeFromISR(ISR).



   
    //Create and start stats task

xTaskCreate( button_task, "button_task", 4096, NULL , 10,&ISR );---------------------------------ISR to resume this created task!!!!---------------



======gpio driver etc for esp8266:
https://github.com/espressif/ESP8266_RTOS_SDK/blob/master/components/freertos/port/esp8266/port.c#L350


timerAttachInterrupt
https://forums.freertos.org/t/basics-timer-interruption-for-triggering-a-task/13402


https://github.com/LetsControltheController/freertos-task-ISR/blob/c7eed0248de01dc98a077812a9d6391414361963/main/main.c
// interrupt service routine, called when the button is pressed
void IRAM_ATTR button_isr_handler(void* arg) {
  
 xTaskResumeFromISR(ISR);
//portYIELD_FROM_ISR(  );
}




https://github.com/DiegoPaezA/ESP32-freeRTOS/blob/master/timerSoftware_1/main.c

void ISR(void *z) {
	int aux = 0;
	xSemaphoreGiveFromISR(smf, &aux); //Entrega el semaforo para su uso

	if (aux) {
		portYIELD_FROM_ISR(); //Fuerza un cambio de contexto
	}
}

void app_main() {
	smf = xSemaphoreCreateBinary(); //crea el semaforo

	tmr = xTimerCreate("tmr_smf", pdMS_TO_TICKS(100), true, 0, ISR); //Crea timer con 100ms de respuesta y autorecarga
	xTimerStart(tmr, pdMS_TO_TICKS(100)); //Inicia el timer

	//Selección de los pines a utilizar
	gpio_pad_select_gpio(GPIO_NUM_2);
	//Configuración de los pines como salida
	gpio_set_direction(GPIO_NUM_2, GPIO_MODE_OUTPUT);

	while (1) {
		if (xSemaphoreTake(smf, portMAX_DELAY)) {
			ESP_LOGI("Timer", "Expiró!"); //Cuando se obtiene el semaforo activará el pin 2 y enviara información al puerto serial.
			gpio_set_level(GPIO_NUM_2, state);
			state = !state;
		}

		vTaskDelay(pdMS_TO_TICKS(10));
	}
}



上下文切换是否可以在 ISR 中发生？
是的。 每个 RTOS 端口都提供宏，以在 ISR 中请求上下文切换。 宏的名称取决于端口（出于历史原因）。 它会 是 portYIELD_FROM_ISR() 或 portEND_SWITCHING_ISR。 具体请参阅 文档页面 查看正在使用的端口的相关信息。

每个官方端口都附有一个 演在 ISR 中进行上下文切换演示应用程序。
可以嵌套中断吗？
这取决于端口。详情请参阅 configKERNEL_INTERRUPT_PRIORITY and configMAX_SYSCALL_INTERRUPT_PRIORITY 


app主动认为调度：vTaskDelay

void vTaskFunction( void * pvParameters )
{
    /* Block for 500ms. */
    const TickType_t xDelay = 500 / portTICK_PERIOD_MS;

     for( ;; )
     {
         /* Simply toggle the LED every 500ms, blocking between each toggle. */
         vToggleLED();
         vTaskDelay( xDelay );
     }
}



vTaskDelay

vTaskSwitchContext
xTaskIncrementTick
xxx vTaskIncrementTick

RTOS 滴答中断发生
RTOS 滴答中断在 TaskA 即将执行 LDI 指令时发生。中断发生时，AVR 微控制器会自动将当前程序计数器 (PC) 置于堆栈上，然后跳转到 RTOS 滴答 ISR 的开始。
https://www.freertos.org/zh-cn-cmn-s/implementation/a00021.html


滴答计数递增
RTOS 函数 vTaskIncrementTick() 在 TaskA 上下文保存后执行。在本例中，假设 tick 计数递增已经导致 TaskB 运行就绪。TaskB 的优先级高于 TaskA，因此 vTaskSwitchContext() 选择 TaskB 作为 ISR 完成时要给予处理时间的任务。



必须恢复 TaskB 上下文。RTOS 宏 portRESTORE_CONTEXT 做的第一件事就是从 TaskB 挂起时获取的拷贝中检索 TaskB 堆栈指针。TaskB 堆栈指针被加载到处理器堆栈指针中，因此现在 AVR 堆栈指向 TaskB 上下文的顶部。
https://www.freertos.org/zh-cn-cmn-s/implementation/a00024.html


发生 RTOS tick 中断时，AVR 自动将 TaskA 返回地址存放在堆栈上，也即 TaskA 中下一条要执行指令的地址。RTOS tick 处理程序更改栈指针，让其当前指向 TaskB 栈。因此，使用 RETI 指令从堆栈弹出的返回地址实际上是 TaskB 在挂起前即将执行的指令的地址。
RTOS tick 中断了 TaskA，但却返回到 TaskB：上下文切换完成！


pc/eip in customized stack:
pxNewTCB = prvCreateTask( pxTaskCode, pcName, uxStackDepth, pvParameters, uxPriority, pxCreatedTask );

  pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters, xRunPrivileged, &( pxNewTCB->xMPUSettings ) );



prvAddNewTaskToReadyList( pxNewTCB );
  /*
 * Place the task represented by pxTCB into the appropriate ready list for
 * the task.  It is inserted at the end of the list.
 */
#define prvAddTaskToReadyList( pxTCB )																\
	traceMOVED_TASK_TO_READY_STATE( pxTCB );														\
	taskRECORD_READY_PRIORITY( ( pxTCB )->uxPriority );												\
	vListInsertEnd( &( pxReadyTasksLists[ ( pxTCB )->uxPriority ] ), &( ( pxTCB )->xStateListItem ) ); \
	tracePOST_MOVED_TASK_TO_READY_STATE( pxTCB )



ontext switching is not directly related to user/kernel space. Context switching relates to the switching between threads/process/interrupt contexts; 
that happens in any RTOS regardless of any concept of user/kernel space or MMU protection.

On processors without a MMU (which is likely most of them used with FreeRTOS, and how it started), things are pretty simple. 
Everyone runs in the same memory space and everyone sort of trusts everyone as there isn’t much to do about it. Task Stacks, 
and the various control blocks can either be created on the ‘Heap’ (either the standard C heap from malloc/free 
(with a wrapping layer to keep them thread safe) or more specialize routines to hand out memory from a specialized heap, 
or with other calls, you can create the needed structures from statically allocated memory.


MPU 是如何实现这一点的？保护与当前执行代码无关联的所有数据是最为行之有效的方法。可以仅使用两个 RTOS 任务（任务 A 和任务 B）来举一个简单的例子。
任务 A 和任务 B 不应彼此交互，但存在一个漏洞 ，其中任务 A 可能意外写入任务 B 偶尔使用的某些数据。覆盖此数据对任务 A 的正确操作无影响。
但任务 B 尝试使用损坏的数据时，任务 B 可能出现异常故障。如果并未配置 MPU 来防止任务 A 写入任务 B 的数据 ，则开发人员可能需要很长时间
才能跟踪到此错误。如果该错误细微不易察觉，或者任务 B 很少使用该数据，那么这一问题将会极难解决。但如果使用 MPU ，错误写入操作即刻出现异常，进而可以确定哪一行代码导致故障。

由于用户可以设置 MPU 区域，防止非特权代码访问 0x0 内存，因此， MPU 甚至可以在某些架构上帮助用户检测空指针偏移。

在您的应用程序中，一组设计良好的 MPU 区域可以显式保护重要的内存区域，以防止特定的问题。例如将缓冲区定位在 MPU 区域的末端来防止缓冲区溢出。
用户还可以将任务栈放置在任何非特权代码无法访问的区域。设置完成后，每个任务必须使用它们自己的某个 MPU 区域，明确通过自行授权获得对其堆栈的访问权限。
如果使用 MPU，用户须真正思考应用程序的结构，以便在任务之间明确分离数据，形成稳健性强且可维护的代码库。



```

- MPU memory protection unit
  
```
FreeRTOS-MPU 移植可以有两种类型的任务：

特权任务：特权任务可以访问整个内存映射。 特权任务可通过使用 xTaskCreate() 或 xTaskCreateRestricted() API 函数创建。
非特权任务：非特权任务只能访问其堆栈。此外， 还可以授予它最多三个用户可定义内存区域的访问权限（每个任务三个） 。非特权任务只能使用 xTaskCreateRestricted() API 创建。 请注意，请勿使用 xTaskCreate() API 创建 非特权任务。
```
