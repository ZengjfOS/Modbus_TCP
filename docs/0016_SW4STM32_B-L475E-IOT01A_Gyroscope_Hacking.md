# SW4STM32 B-L475E-IOT01A Gyroscope Hakcing

## Gyroscope Init

```C

/**
 * typedef struct
 * {  
 *   void       (*Init)(uint16_t);
 *   void       (*DeInit)(void); 
 *   uint8_t    (*ReadID)(void);
 *   void       (*Reset)(void);
 *   void       (*LowPower)(uint16_t);   
 *   void       (*ConfigIT)(uint16_t); 
 *   void       (*EnableIT)(uint8_t);
 *   void       (*DisableIT)(uint8_t);  
 *   uint8_t    (*ITStatus)(uint16_t, uint16_t);   
 *   void       (*ClearIT)(uint16_t, uint16_t); 
 *   void       (*FilterConfig)(uint8_t);  
 *   void       (*FilterCmd)(uint8_t);  
 *   void       (*GetXYZ)(float *);
 * }GYRO_DrvTypeDef;
 */
static GYRO_DrvTypeDef *GyroscopeDrv;

/**
  * @brief  Initialize Gyroscope.
  * @retval GYRO_OK or GYRO_ERROR
  */
uint8_t BSP_GYRO_Init(void)
{  
  uint8_t ret = GYRO_ERROR;
  uint16_t ctrl = 0x0000;
  /**
   * typedef struct
   * {
   *   uint8_t Power_Mode;                         /* Power-down/Sleep/Normal Mode */
   *   uint8_t Output_DataRate;                    /* OUT data rate */
   *   uint8_t Axes_Enable;                        /* Axes enable */
   *   uint8_t Band_Width;                         /* Bandwidth selection */
   *   uint8_t BlockData_Update;                   /* Block Data Update */
   *   uint8_t Endianness;                         /* Endian Data selection */
   *   uint8_t Full_Scale;                         /* Full Scale selection */
   * }GYRO_InitTypeDef;
   */
  GYRO_InitTypeDef LSM6DSL_InitStructure;

  /**
   * GYRO_DrvTypeDef Lsm6dslGyroDrv =
   * {
   *   LSM6DSL_GyroInit,
   *   LSM6DSL_GyroDeInit,
   *   LSM6DSL_GyroReadID,
   *   0,
   *   LSM6DSL_GyroLowPower,
   *   0,
   *   0,
   *   0,
   *   0,
   *   0,
   *   0,
   *   0,
   *   LSM6DSL_GyroReadXYZAngRate                       <-----   void       (*GetXYZ)(float *);
   * };
   */
  if(Lsm6dslGyroDrv.ReadID() != LSM6DSL_ACC_GYRO_WHO_AM_I)
  {
    ret = GYRO_ERROR;
  }
  else
  {
    /* Initialize the gyroscope driver structure */
    GyroscopeDrv = &Lsm6dslGyroDrv;

    /* Configure Mems : data rate, power mode, full scale and axes */
    LSM6DSL_InitStructure.Power_Mode = 0;
    LSM6DSL_InitStructure.Output_DataRate = LSM6DSL_ODR_52Hz;
    LSM6DSL_InitStructure.Axes_Enable = 0;
    LSM6DSL_InitStructure.Band_Width = 0;
    LSM6DSL_InitStructure.BlockData_Update = LSM6DSL_BDU_BLOCK_UPDATE;
    LSM6DSL_InitStructure.Endianness = 0;
    LSM6DSL_InitStructure.Full_Scale = LSM6DSL_GYRO_FS_2000;                     <----------设置量程

    /* Configure MEMS: data rate, full scale  */
    ctrl = (LSM6DSL_InitStructure.Full_Scale | LSM6DSL_InitStructure.Output_DataRate);

    /* Configure MEMS: BDU and Auto-increment for multi read/write */
    ctrl |= ((LSM6DSL_InitStructure.BlockData_Update | LSM6DSL_ACC_GYRO_IF_INC_ENABLED) << 8);

    /* Initialize component */
    /**
     * void LSM6DSL_GyroInit(uint16_t InitStruct)
     * {  
     *   uint8_t ctrl = 0x00;
     *   uint8_t tmp;
     * 
     *   /* Read CTRL2_G */
     *   tmp = SENSOR_IO_Read(LSM6DSL_ACC_GYRO_I2C_ADDRESS_LOW, LSM6DSL_ACC_GYRO_CTRL2_G);
     * 
     *   /* Write value to GYRO MEMS CTRL2_G register: FS and Data Rate */
     *   ctrl = (uint8_t) InitStruct;
     *   tmp &= ~(0xFC);
     *   tmp |= ctrl;
     *   SENSOR_IO_Write(LSM6DSL_ACC_GYRO_I2C_ADDRESS_LOW, LSM6DSL_ACC_GYRO_CTRL2_G, tmp);
     * 
     *   /* Read CTRL3_C */
     *   tmp = SENSOR_IO_Read(LSM6DSL_ACC_GYRO_I2C_ADDRESS_LOW, LSM6DSL_ACC_GYRO_CTRL3_C);
     * 
     *   /* Write value to GYRO MEMS CTRL3_C register: BDU and Auto-increment */
     *   ctrl = ((uint8_t) (InitStruct >> 8));
     *   tmp &= ~(0x44);
     *   tmp |= ctrl; 
     *   SENSOR_IO_Write(LSM6DSL_ACC_GYRO_I2C_ADDRESS_LOW, LSM6DSL_ACC_GYRO_CTRL3_C, tmp);
     * }
     */
    GyroscopeDrv->Init(ctrl);
    
    ret = GYRO_OK;
  }
  
  return ret;
}
```

## Gyroscope GetXYZ

```C
void BSP_GYRO_GetXYZ(float* pfData)
{
  if(GyroscopeDrv != NULL)
  {
    if(GyroscopeDrv->GetXYZ!= NULL)
    {
      GyroscopeDrv->GetXYZ(pfData);
    }                  |
  }                    |
}                      |
                       V
void LSM6DSL_GyroReadXYZAngRate(float *pfData)
{
  int16_t pnRawData[3];
  uint8_t ctrlg= 0;
  uint8_t buffer[6];
  uint8_t i = 0;
  float sensitivity = 0;
  
  /* Read the gyro control register content */
  ctrlg = SENSOR_IO_Read(LSM6DSL_ACC_GYRO_I2C_ADDRESS_LOW, LSM6DSL_ACC_GYRO_CTRL2_G);
  
  /* Read output register X, Y & Z acceleration */
  SENSOR_IO_ReadMultiple(LSM6DSL_ACC_GYRO_I2C_ADDRESS_LOW, LSM6DSL_ACC_GYRO_OUTX_L_G, buffer, 6);
  
  for(i=0; i<3; i++)
  {
    pnRawData[i]=((((uint16_t)buffer[2*i+1]) << 8) + (uint16_t)buffer[2*i]);
  }
  
  /* Normal mode */
  /* Switch the sensitivity value set in the CRTL2_G */
  switch(ctrlg & 0x0C)
  {
  case LSM6DSL_GYRO_FS_245:
    sensitivity = LSM6DSL_GYRO_SENSITIVITY_245DPS;
    break;
  case LSM6DSL_GYRO_FS_500:
    sensitivity = LSM6DSL_GYRO_SENSITIVITY_500DPS;
    break;
  case LSM6DSL_GYRO_FS_1000:
    sensitivity = LSM6DSL_GYRO_SENSITIVITY_1000DPS;
    break;
  case LSM6DSL_GYRO_FS_2000:
    // #define LSM6DSL_GYRO_SENSITIVITY_2000DPS ((float)70.00f) /**< Sensitivity value for 2000 dps full scale [mdps/LSB] */ 
    sensitivity = LSM6DSL_GYRO_SENSITIVITY_2000DPS;
    break;    
  }
  
  /* Obtain the mg value for the three axis */
  for(i=0; i<3; i++)
  {
    pfData[i]=( float )(pnRawData[i] * sensitivity);
  }
}
```
