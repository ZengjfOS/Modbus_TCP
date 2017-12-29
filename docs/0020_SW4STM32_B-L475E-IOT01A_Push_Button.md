# SW4STM32 B-L475E-IOT01A Push Button

## Push Button

```C
int main(void)
{
  ...
  BSP_PB_Init(BUTTON_USER, BUTTON_MODE_EXTI);    -------------------------------+--------+
  ...                  |           |                                            |        |
}                      |           |                                            |        |
                       |           |                                            |        |
typedef enum           |           |                                            |        |
{                      |           |                                            |        |
  BUTTON_USER = 0 <----+           |                                            |        |
}Button_TypeDef;                   |                                            |        |
                                   |                                            |        |
typedef enum                       |                                            |        |
{                                  |                                            |        |
  BUTTON_MODE_GPIO = 0,            |                                            |        |
  BUTTON_MODE_EXTI = 1   <---------+                                            |        |
}ButtonMode_TypeDef;                                                            |        |
                                                                                |        |
void BSP_PB_Init(Button_TypeDef Button, ButtonMode_TypeDef ButtonMode)  <-------+        |
{                                                                                        |
  GPIO_InitTypeDef gpio_init_structure;                                                  |
                                                                                         |
  /* Enable the BUTTON clock */                                                          |
  USER_BUTTON_GPIO_CLK_ENABLE();                                                         |
                                                                                         |
  if(ButtonMode == BUTTON_MODE_GPIO)                                                     |
  {                                                                                      |
    /* Configure Button pin as input */                                                  |
    gpio_init_structure.Pin = BUTTON_PIN[Button];          ----------------+             |
    gpio_init_structure.Mode = GPIO_MODE_INPUT;                            |             |
    gpio_init_structure.Pull = GPIO_PULLUP;                                |             |
    gpio_init_structure.Speed = GPIO_SPEED_FREQ_HIGH;                      |             |
    HAL_GPIO_Init(BUTTON_PORT[Button], &gpio_init_structure);  ------------*-+           |
  }                                                                        | |           |
                                                                           | |           |
  if(ButtonMode == BUTTON_MODE_EXTI)                                       | |           |
  {                                                                        | |           |
    /* Configure Button pin as input with External interrupt */            | |           |
    gpio_init_structure.Pin = BUTTON_PIN[Button];                          | |           |
    gpio_init_structure.Pull = GPIO_PULLUP;                                | |           |
    gpio_init_structure.Speed = GPIO_SPEED_FREQ_VERY_HIGH;                 | |           |
                                                                           | |           |
    gpio_init_structure.Mode = GPIO_MODE_IT_RISING;                        | |           |
                                                                           | |           |
    HAL_GPIO_Init(BUTTON_PORT[Button], &gpio_init_structure);              | |           |
                                                                           | |           |
    /* Enable and set Button EXTI Interrupt to the lowest priority */      | |           |
    HAL_NVIC_SetPriority((IRQn_Type)(BUTTON_IRQn[Button]), 0x0F, 0x00);    | |           |
    HAL_NVIC_EnableIRQ((IRQn_Type)(BUTTON_IRQn[Button]));                  | |           |
  }                                                                        | |           |
}                                                                          | |           |
                   v-------------------------------------------------------+ |           |
const uint16_t BUTTON_PIN[BUTTONn] = {USER_BUTTON_PIN};                      |           |
                 v---------------------------^                               |           |
#define USER_BUTTON_PIN                   GPIO_PIN_13                        |           |
                     v-------------------------------------------------------+           |
GPIO_TypeDef* BUTTON_PORT[BUTTONn] = {USER_BUTTON_GPIO_PORT};                            |
                 v----------------------------^                                          |
#define USER_BUTTON_GPIO_PORT             GPIOC                                          |
                                                                                         |
void             BSP_PB_Init(Button_TypeDef Button, ButtonMode_TypeDef ButtonMode); <----+
void             BSP_PB_DeInit(Button_TypeDef Button);     ------+
uint32_t         BSP_PB_GetState(Button_TypeDef Button);   ------*-----------+
                                                                 |           |
uint32_t BSP_PB_GetState(Button_TypeDef Button)       <----------+           |
{                                                                            |
  return HAL_GPIO_ReadPin(BUTTON_PORT[Button], BUTTON_PIN[Button]);          |
}                                                                            |
                                                                             |
GPIO_PinState HAL_GPIO_ReadPin(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin) <-----+
{
  GPIO_PinState bitstatus;

  /* Check the parameters */
  assert_param(IS_GPIO_PIN(GPIO_Pin));

  if((GPIOx->IDR & GPIO_Pin) != (uint32_t)GPIO_PIN_RESET)
  {
    bitstatus = GPIO_PIN_SET;
  }
  else
  {
    bitstatus = GPIO_PIN_RESET;
  }
  return bitstatus;
}


```

## Interrupt

```C
g_pfnVectors:
    .word    EXTI15_10_IRQHandler    -----------+
                                                |
void EXTI15_10_IRQHandler(void)      <----------+
{
  HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);               ---------------------------+
}                              |                                                  |
             v-----------------+                                                  |
#define GPIO_PIN_13                ((uint16_t)0x2000)  /* Pin 13 selected   */    |
                                                                                  |
void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin)       <--------------------------+
{
  /* EXTI line interrupt detected */
  if(__HAL_GPIO_EXTI_GET_IT(GPIO_Pin) != RESET)
  {
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin);
    HAL_GPIO_EXTI_Callback(GPIO_Pin);           ----------------+
  }                                                             |
}                                                               |
                                                                |
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)  <---------------+
{
  switch (GPIO_Pin)
  {
    case (GPIO_PIN_13): /* From mxconstants.h BUTTON_EXTI13_Pin */
    {
      Button_ISR();       --------------------+
      break;                                  |
    }                                         |
                                              |
    default:                                  |
    {                                         |
      break;                                  |
    }                                         |
  }                                           |
}                                             |
                                              |
static void Button_ISR(void)     <------------+
{
  button_flags++;                ----------------------+
}                                                      |
                                                       |
uint8_t Button_WaitForPush(uint32_t delay)             |
{                                                      |
  uint32_t time_out = HAL_GetTick()+delay;             |
  do                                                   |
  {                                                    |
    if (button_flags > 1)                              |
    {                                                  |
      button_flags = 0;            <-------------------+
      return BP_MULTIPLE_PUSH;
    }

    if (button_flags == 1)
    {
      button_flags = 0;
      return BP_SINGLE_PUSH;
    }
  }
  while( HAL_GetTick() < time_out);
  return BP_NOT_PUSHED;
}
```
