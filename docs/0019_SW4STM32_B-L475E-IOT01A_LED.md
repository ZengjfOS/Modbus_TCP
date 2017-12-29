# SW4STM32 B-L475E-IOT01A LED

```C
int main(void)
{
  ...
  BSP_LED_Init(LED_GREEN);    ------------------+--------------------------------------------------------------+
  ...                 |                         |                                                              |
}                     |                         |                                                              |
                      |                         |                                                              |
typedef enum          |                         |                                                              |
{                     |                         |                                                              |
LED2 = 0,             |                         |                                                              |
LED_GREEN = LED2, <---+                         |                                                              |
}Led_TypeDef;                                   |                                                              |
                                                |                                                              |
void BSP_LED_Init(Led_TypeDef Led)   <----------+                                                              |
{                                                                                                              |
  GPIO_InitTypeDef  gpio_init_structure;                                                                       |
                                                                                                               |
  LEDx_GPIO_CLK_ENABLE(Led);    ---------------------------+                                                   |
  /* Configure the GPIO_LED pin */                         |                                                   |
  gpio_init_structure.Pin   = GPIO_PIN[Led];       --------*-----------------------------------------------+   |
  gpio_init_structure.Mode  = GPIO_MODE_OUTPUT_PP;         |                                               |   |
  // gpio_init_structure.Pull  = GPIO_NOPULL;              |                                               |   |
  gpio_init_structure.Pull  = GPIO_PULLUP;                 |                                               |   |
  gpio_init_structure.Speed = GPIO_SPEED_FREQ_HIGH;        |                                               |   |
                                                           |                                               |   |
  HAL_GPIO_Init(GPIO_PORT[Led], &gpio_init_structure);     |           ------------------------------------*-+ |
}                    +-------------------------------------+                                               | | |
                     V                                                                                     | | |
#define LEDx_GPIO_CLK_ENABLE(__INDEX__)  do{if((__INDEX__) == 0) LED2_GPIO_CLK_ENABLE();}while(0)          | | |
                     v---------------------------------------------------^                                 | | |
#define LED2_GPIO_CLK_ENABLE()           __HAL_RCC_GPIOB_CLK_ENABLE()                                      | | |
                     v----------------------------------^                                                  | | |
#define __HAL_RCC_GPIOB_CLK_ENABLE()           do { \                                                      | | |
                                                 __IO uint32_t tmpreg; \                                   | | |
                                                 SET_BIT(RCC->AHB2ENR, RCC_AHB2ENR_GPIOBEN); \             | | |
                                                 /* Delay after an RCC peripheral clock enabling */ \      | | |
                                                 tmpreg = READ_BIT(RCC->AHB2ENR, RCC_AHB2ENR_GPIOBEN); \   | | |
                                                 UNUSED(tmpreg); \                                         | | |
                                               } while(0)                                                  | | |
                    v--------------------------------------------------------------------------------------+ | |
const uint32_t GPIO_PIN[LEDn] = {LED2_PIN};    -------------+                                                | |
         v---------------^                                  |                                                | |
#define LEDn                             ((uint8_t)1)       |                                                | |
                                                            |                                                | |
#define LED2_PIN                         GPIO_PIN_14 <------+                                                | |
                                                                                                             | |
GPIO_TypeDef* GPIO_PORT[LEDn] = {LED2_GPIO_PORT};                                <---------------------------+ |
                v----------------------^                                                                     | |
#define LED2_GPIO_PORT                   GPIOB                                   ----------------------------*-*-+
                                                                                                             | | |
void HAL_GPIO_Init(GPIO_TypeDef  *GPIOx, GPIO_InitTypeDef *GPIO_Init)            <---------------------------+ | |
{                                                                                                              | |
  uint32_t position = 0x00;                                                                                    | |
  uint32_t iocurrent = 0x00;                                                                                   | |
  uint32_t temp = 0x00;                                                                                        | |
                                                                                                               | |
  /* Check the parameters */                                                                                   | |
  assert_param(IS_GPIO_ALL_INSTANCE(GPIOx));                                                                   | |
  assert_param(IS_GPIO_PIN(GPIO_Init->Pin));                                                                   | |
  assert_param(IS_GPIO_MODE(GPIO_Init->Mode));                                                                 | |
  assert_param(IS_GPIO_PULL(GPIO_Init->Pull));                                                                 | |
                                                                                                               | |
  /* Configure the port pins */                                                                                | |
  while (((GPIO_Init->Pin) >> position) != RESET)                                                              | |
  {                                                                                                            | |
    /* Get current io position */                                                                              | |
    iocurrent = (GPIO_Init->Pin) & (1U << position);                                                           | |
                                                                                                               | |
    if(iocurrent)                                                                                              | |
    {                                                                                                          | |
      /*--------------------- GPIO Mode Configuration ------------------------*/                               | |
      /* In case of Alternate function mode selection */                                                       | |
      if((GPIO_Init->Mode == GPIO_MODE_AF_PP) || (GPIO_Init->Mode == GPIO_MODE_AF_OD))                         | |
      {                                                                                                        | |
        /* Check the Alternate function parameters */                                                          | |
        assert_param(IS_GPIO_AF_INSTANCE(GPIOx));                                                              | |
        assert_param(IS_GPIO_AF(GPIO_Init->Alternate));                                                        | |
                                                                                                               | |
        /* Configure Alternate function mapped with the current IO */                                          | |
        temp = GPIOx->AFR[position >> 3];                                                                      | |
        temp &= ~((uint32_t)0xF << ((uint32_t)(position & (uint32_t)0x07) * 4)) ;                              | |
        temp |= ((uint32_t)(GPIO_Init->Alternate) << (((uint32_t)position & (uint32_t)0x07) * 4));             | |
        GPIOx->AFR[position >> 3] = temp;                                                                      | |
      }                                                                                                        | |
                                                                                                               | |
      /* Configure IO Direction mode (Input, Output, Alternate or Analog) */                                   | |
      temp = GPIOx->MODER;                                                                                     | |
      temp &= ~(GPIO_MODER_MODE0 << (position * 2));                                                           | |
      temp |= ((GPIO_Init->Mode & GPIO_MODE) << (position * 2));                                               | |
      GPIOx->MODER = temp;                                                                                     | |
                                                                                                               | |
      /* In case of Output or Alternate function mode selection */                                             | |
      if((GPIO_Init->Mode == GPIO_MODE_OUTPUT_PP) || (GPIO_Init->Mode == GPIO_MODE_AF_PP) ||                   | |
         (GPIO_Init->Mode == GPIO_MODE_OUTPUT_OD) || (GPIO_Init->Mode == GPIO_MODE_AF_OD))                     | |
      {                                                                                                        | |
        /* Check the Speed parameter */                                                                        | |
        assert_param(IS_GPIO_SPEED(GPIO_Init->Speed));                                                         | |
        /* Configure the IO Speed */                                                                           | |
        temp = GPIOx->OSPEEDR;                                                                                 | |
        temp &= ~(GPIO_OSPEEDR_OSPEED0 << (position * 2));                                                     | |
        temp |= (GPIO_Init->Speed << (position * 2));                                                          | |
        GPIOx->OSPEEDR = temp;                                                                                 | |
                                                                                                               | |
        /* Configure the IO Output Type */                                                                     | |
        temp = GPIOx->OTYPER;                                                                                  | |
        temp &= ~(GPIO_OTYPER_OT0 << position) ;                                                               | |
        temp |= (((GPIO_Init->Mode & GPIO_OUTPUT_TYPE) >> 4) << position);                                     | |
        GPIOx->OTYPER = temp;                                                                                  | |
      }                                                                                                        | |
                                                                                                               | |
#if defined(STM32L471xx) || defined(STM32L475xx) || defined(STM32L476xx) || defined(STM32L485xx) || defined(STM32L486xx)
                                                                                                               | |
      /* In case of Analog mode, check if ADC control mode is selected */                                      | |
      if((GPIO_Init->Mode & GPIO_MODE_ANALOG) == GPIO_MODE_ANALOG)                                             | |
      {                                                                                                        | |
        /* Configure the IO Output Type */                                                                     | |
        temp = GPIOx->ASCR;                                                                                    | |
        temp &= ~(GPIO_ASCR_ASC0 << position) ;                                                                | |
        temp |= (((GPIO_Init->Mode & ANALOG_MODE) >> 3) << position);                                          | |
        GPIOx->ASCR = temp;                                                                                    | |
      }                                                                                                        | |
                                                                                                               | |
#endif /* STM32L471xx || STM32L475xx || STM32L476xx || STM32L485xx || STM32L486xx */                           | |
                                                                                                               | |
      /* Activate the Pull-up or Pull down resistor for the current IO */                                      | |
      temp = GPIOx->PUPDR;                                                                                     | |
      temp &= ~(GPIO_PUPDR_PUPD0 << (position * 2));                                                           | |
      temp |= ((GPIO_Init->Pull) << (position * 2));                                                           | |
      GPIOx->PUPDR = temp;                                                                                     | |
                                                                                                               | |
      /*--------------------- EXTI Mode Configuration ------------------------*/                               | |
      /* Configure the External Interrupt or event for the current IO */                                       | |
      if((GPIO_Init->Mode & EXTI_MODE) == EXTI_MODE)                                                           | |
      {                                                                                                        | |
        /* Enable SYSCFG Clock */                                                                              | |
        __HAL_RCC_SYSCFG_CLK_ENABLE();                                                                         | |
                                                                                                               | |
        temp = SYSCFG->EXTICR[position >> 2];                                                                  | |
        temp &= ~(((uint32_t)0x0F) << (4 * (position & 0x03)));                                                | |
        temp |= (GPIO_GET_INDEX(GPIOx) << (4 * (position & 0x03)));                                            | |
        SYSCFG->EXTICR[position >> 2] = temp;                                                                  | |
                                                                                                               | |
        /* Clear EXTI line configuration */                                                                    | |
        temp = EXTI->IMR1;                                                                                     | |
        temp &= ~((uint32_t)iocurrent);                                                                        | |
        if((GPIO_Init->Mode & GPIO_MODE_IT) == GPIO_MODE_IT)                                                   | |
        {                                                                                                      | |
          temp |= iocurrent;                                                                                   | |
        }                                                                                                      | |
        EXTI->IMR1 = temp;                                                                                     | |
                                                                                                               | |
        temp = EXTI->EMR1;                                                                                     | |
        temp &= ~((uint32_t)iocurrent);                                                                        | |
        if((GPIO_Init->Mode & GPIO_MODE_EVT) == GPIO_MODE_EVT)                                                 | |
        {                                                                                                      | |
          temp |= iocurrent;                                                                                   | |
        }                                                                                                      | |
        EXTI->EMR1 = temp;                                                                                     | |
                                                                                                               | |
        /* Clear Rising Falling edge configuration */                                                          | |
        temp = EXTI->RTSR1;                                                                                    | |
        temp &= ~((uint32_t)iocurrent);                                                                        | |
        if((GPIO_Init->Mode & RISING_EDGE) == RISING_EDGE)                                                     | |
        {                                                                                                      | |
          temp |= iocurrent;                                                                                   | |
        }                                                                                                      | |
        EXTI->RTSR1 = temp;                                                                                    | |
                                                                                                               | |
        temp = EXTI->FTSR1;                                                                                    | |
        temp &= ~((uint32_t)iocurrent);                                                                        | |
        if((GPIO_Init->Mode & FALLING_EDGE) == FALLING_EDGE)                                                   | |
        {                                                                                                      | |
          temp |= iocurrent;                                                                                   | |
        }                                                                                                      | |
        EXTI->FTSR1 = temp;                                                                                    | |
      }                                                                                                        | |
    }                                                                                                          | |
                                                                                                               | |
    position++;                                                                                                | |
  }                                                                                                            | |
}                                                                                                              | |
                                                                                                               | |
void             BSP_LED_Init(Led_TypeDef Led);      <---------------------------------------------------------+ |
void             BSP_LED_On(Led_TypeDef Led);        ------+                                                     |
void             BSP_LED_Off(Led_TypeDef Led);       ------*----------+                                          |
void             BSP_LED_Toggle(Led_TypeDef Led);    ------*----------*---+                                      |
                                                           |          |   |                                      |
void BSP_LED_On(Led_TypeDef Led)               <-----------+          |   |                                      |
{                                                                     |   |                                      |
  HAL_GPIO_WritePin(GPIO_PORT[Led], GPIO_PIN[Led], GPIO_PIN_SET);  ---*---*-----------------+                    |
}                                                                     |   |                 |                    |
                                                                      |   |                 |                    |
void BSP_LED_Off(Led_TypeDef Led)              <----------------------+   |                 |                    |
{                                                                         |                 |                    |
  HAL_GPIO_WritePin(GPIO_PORT[Led], GPIO_PIN[Led], GPIO_PIN_RESET);       |                 |                    |
}                                                                         |                 |                    |
                                                                          |                 |                    |
void BSP_LED_Toggle(Led_TypeDef Led)           <--------------------------+                 |                    |
{                                                                                           |                    |
  HAL_GPIO_TogglePin(GPIO_PORT[Led], GPIO_PIN[Led]);                 -----------------------*-+                  |
}                                                                                           | |                  |
                                                                                            | |                  |
void HAL_GPIO_WritePin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, GPIO_PinState PinState)  <---+ |                  |
{                                                                                             |                  |
  /* Check the parameters */                                                                  |                  |
  assert_param(IS_GPIO_PIN(GPIO_Pin));                                                        |                  |
  assert_param(IS_GPIO_PIN_ACTION(PinState));                                                 |                  |
                                                                                              |                  |
  if(PinState != GPIO_PIN_RESET)                                                              |                  |
  {                                                                                           |                  |
    GPIOx->BSRR = (uint32_t)GPIO_Pin;                                                         |                  |
  }                                                                                           |                  |
  else                                                                                        |                  |
  {                                                                                           |                  |
    GPIOx->BRR = (uint32_t)GPIO_Pin;                                                          |                  |
  }                                                                                           |                  |
}                                                                                             |                  |
                                                                                              |                  |
void HAL_GPIO_TogglePin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)    <--------------------------+                  |
{                                                                                                                |
  /* Check the parameters */                                                                                     |
  assert_param(IS_GPIO_PIN(GPIO_Pin));                                                                           |
                                                                                                                 |
  GPIOx->ODR ^= GPIO_Pin;                                                                                        |
}                                                                                                                |
          v------------------------------------------------------------------------------------------------------+
#define GPIOB               ((GPIO_TypeDef *) GPIOB_BASE)          ----------------------------------------------+
            v------------------------------------^                                                               |
#define GPIOB_BASE            (APB2PERIPH_BASE + 0x00000C00U)                                                    |
                                                                                                                 |
typedef struct                                                                                                   |
{                                                                                                                |
  __IO uint32_t MODER;       /*!< GPIO port mode register,               Address offset: 0x00      */            |
  __IO uint32_t OTYPER;      /*!< GPIO port output type register,        Address offset: 0x04      */            |
  __IO uint32_t OSPEEDR;     /*!< GPIO port output speed register,       Address offset: 0x08      */            |
  __IO uint32_t PUPDR;       /*!< GPIO port pull-up/pull-down register,  Address offset: 0x0C      */            |
  __IO uint32_t IDR;         /*!< GPIO port input data register,         Address offset: 0x10      */            |
  __IO uint32_t ODR;         /*!< GPIO port output data register,        Address offset: 0x14      */            |
  __IO uint32_t BSRR;        /*!< GPIO port bit set/reset  register,     Address offset: 0x18      */            |
  __IO uint32_t LCKR;        /*!< GPIO port configuration lock register, Address offset: 0x1C      */            |
  __IO uint32_t AFR[2];      /*!< GPIO alternate function registers,     Address offset: 0x20-0x24 */            |
  __IO uint32_t BRR;         /*!< GPIO Bit Reset register,               Address offset: 0x28      */            |
  __IO uint32_t ASCR;        /*!< GPIO analog switch control register,   Address offset: 0x2C     */             |
                                                                                                                 |
} GPIO_TypeDef;        <-----------------------------------------------------------------------------------------+
```
