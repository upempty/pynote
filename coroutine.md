

- coroutine  
  https://www.linuxhowtos.org/C_C++/coroutines.htm  

  https://github.com/maniacbug/FreeRTOS/blob/master/croutine.h#L267  
  
  https://www.freertos.org/zh-cn-cmn-s/crcreate.html
  

  ```

 // Co-routine to be created.
 void vFlashCoRoutine( CoRoutineHandle_t xHandle, UBaseType_t uxIndex )
 {
 // Variables in co-routines must be declared static if they must 
 // maintain value across a blocking call. This may not be necessary 
 // for const variables.
 static const char cLedToFlash[ 2 ] = { 5, 6 };
 static const TickType_t uxFlashRates[ 2 ] = { 200, 400 };

     // Must start every co-routine with a call to crSTART();
     crSTART( xHandle );

     for( ;; )
     {
         // This co-routine just delays for a fixed period, then toggles
         // an LED.  Two co-routines are created using this function, so
         // the uxIndex parameter is used to tell the co-routine which
         // LED to flash and how long to delay.  This assumes xQueue has
         // already been created.
         vParTestToggleLED( cLedToFlash[ uxIndex ] );


        ///
        #define crDELAY( xHandle, xTicksToDelay )												\------------------------------------------------
	if( ( xTicksToDelay ) > 0 )															\
	{																					\
		vCoRoutineAddToDelayedList( ( xTicksToDelay ), NULL );							\
	}																					\
	crSET_STATE0( ( xHandle ) );
        ///

        
         crDELAY( xHandle, uxFlashRates[ uxIndex ] );------------push to delay list(like sleep time), so it will -------#define crSET_STATE0( xHandle ) ( ( corCRCB * )( xHandle ) )->uxState = (__LINE__ * 2); return; case (__LINE__ * 2):


         ===return then go to next schedule after coroutine call() finished with return.
	 ===!!!!next executing position address [Here still in the loop, just not iteral] for next coroutine scheduling. Key point is to routine/function level known address for next coroutine.!!!!
        ///


void vCoRoutineSchedule( void )------------------------------------------
{
	/* See if any co-routines readied by events need moving to the ready lists. */
	prvCheckPendingReadyList();

	/* See if any delayed co-routines have timed out. */
	prvCheckDelayedList();

	/* Find the highest priority queue that contains ready co-routines. */
	while( listLIST_IS_EMPTY( &( pxReadyCoRoutineLists[ uxTopCoRoutineReadyPriority ] ) ) )
	{
		if( uxTopCoRoutineReadyPriority == 0 )
		{
			/* No more co-routines to check. */
			return;
		}
		--uxTopCoRoutineReadyPriority;
	}

	/* listGET_OWNER_OF_NEXT_ENTRY walks through the list, so the co-routines
	 of the	same priority get an equal share of the processor time. */
	listGET_OWNER_OF_NEXT_ENTRY( pxCurrentCoRoutine, &( pxReadyCoRoutineLists[ uxTopCoRoutineReadyPriority ] ) );

	/* Call the co-routine. */
	( pxCurrentCoRoutine->pxCoRoutineFunction )( pxCurrentCoRoutine, pxCurrentCoRoutine->uxIndex );---------------------------------------vFlashCoRoutine finished, then go to next for coroutine schduler vCoRoutineSchedule.

	return;
}

    
        ///


        ////
         // This idle task hook will schedule a co-routine each time it is called.
 // The rest of the idle task will execute between co-routine calls.
 void vApplicationIdleHook( void )
 {
	vCoRoutineSchedule();------------------------------------------------------done in idle thread.!!!!!!!!!
 }
 
        ////
     }

     // Must end every co-routine with a call to crEND();
     crEND();
 }

 // Function that creates two co-routines.
 void vOtherFunction( void )
 {
 unsigned char ucParameterToPass;
 TaskHandle_t xHandle;
		
     // Create two co-routines at priority 0.  The first is given index 0
     // so (from the code above) toggles LED 5 every 200 ticks.  The second
     // is given index 1 so toggles LED 6 every 400 ticks.
     for( uxIndex = 0; uxIndex < 2; uxIndex++ )
     {
         xCoRoutineCreate( vFlashCoRoutine, 0, uxIndex );
     }  
 }

 
  ```
