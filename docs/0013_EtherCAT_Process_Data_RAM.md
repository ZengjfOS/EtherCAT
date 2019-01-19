# EtherCAT Process Data RAM

## 参考文档

* [Ethercat解析（九）之过程数据](https://blog.csdn.net/absinjun/article/details/81603644)

## 说明

* 由参考文档可知，过程数据，可以认为是CANOpen PDO数据集合；
* PDI中断解析源代码：
  ```C
  void PDI_Isr(void)
  {
      if(bEscIntEnabled)
      {
          /* get the AL event register */
          UINT16  ALEvent = HW_GetALEventRegister_Isr();
          ALEvent = SWAPWORD(ALEvent);
  
          if ( ALEvent & PROCESS_OUTPUT_EVENT )
          {
              if(bDcRunning && bDcSyncActive)
              {
                  /* Reset SM/Sync0 counter. Will be incremented on every Sync0 event*/
                  u16SmSync0Counter = 0;
              }
              if(sSyncManOutPar.u16SmEventMissedCounter > 0)
                  sSyncManOutPar.u16SmEventMissedCounter--;
  
  
  /*ECATCHANGE_START(V5.11) ECAT6*/
              //calculate the bus cycle time if required
              HandleBusCycleCalculation();
  /*ECATCHANGE_END(V5.11) ECAT6*/
  
          /* Outputs were updated, set flag for watchdog monitoring */
          bEcatFirstOutputsReceived = TRUE;
  
  
          /*
              handle output process data event
          */
          if ( bEcatOutputUpdateRunning )
          {
              /* slave is in OP, update the outputs */
              PDO_OutputMapping();
          }
          else
          {
              /* Just acknowledge the process data event in the INIT,PreOP and SafeOP state */
              HW_EscReadWordIsr(u16dummy,nEscAddrOutputData);
              HW_EscReadWordIsr(u16dummy,(nEscAddrOutputData+nPdOutputSize-2));
          }
          }
  
  /*ECATCHANGE_START(V5.11) ECAT4*/
          if (( ALEvent & PROCESS_INPUT_EVENT ) && (nPdOutputSize == 0))
          {
              //calculate the bus cycle time if required
              HandleBusCycleCalculation();
          }
  /*ECATCHANGE_END(V5.11) ECAT4*/
  
          /*
              Call ECAT_Application() in SM Sync mode
          */
          if (sSyncManOutPar.u16SyncType == SYNCTYPE_SM_SYNCHRON)
          {
              /* The Application is synchronized to process data Sync Manager event*/
              ECAT_Application();
          }
  
      if ( bEcatInputUpdateRunning 
  /*ECATCHANGE_START(V5.11) ESM7*/
         && ((sSyncManInPar.u16SyncType == SYNCTYPE_SM_SYNCHRON) || (sSyncManInPar.u16SyncType == SYNCTYPE_SM2_SYNCHRON))
  /*ECATCHANGE_END(V5.11) ESM7*/
          )
      {
          /* EtherCAT slave is at least in SAFE-OPERATIONAL, update inputs */
          PDO_InputMapping();
      }
  
      /*
        Check if cycle exceed
      */
      /*if next SM event was triggered during runtime increment cycle exceed counter*/
      ALEvent = HW_GetALEventRegister_Isr();
      ALEvent = SWAPWORD(ALEvent);
  
      if ( ALEvent & PROCESS_OUTPUT_EVENT )
      {
          sSyncManOutPar.u16CycleExceededCounter++;
          sSyncManInPar.u16CycleExceededCounter = sSyncManOutPar.u16CycleExceededCounter;
  
        /* Acknowledge the process data event*/
              HW_EscReadWordIsr(u16dummy,nEscAddrOutputData);
              HW_EscReadWordIsr(u16dummy,(nEscAddrOutputData+nPdOutputSize-2));
      }
      } //if(bEscIntEnabled)
  }
  ```
  
