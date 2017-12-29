# SW4STM32 B-L475E-IOT01A Printf Hacking

[code/syscalls.c](code/syscalls.c)

```C
static UART_HandleTypeDef console_uart;          ---------------------------------+
                                                                                  |
int main(void)                                                                    |
{                                                                                 |
    ...                                                                           |
    Console_UART_Init();      --------------------------+                         |
    ...                                                 |                         |
}                                                       |                         |
                                                        |                         |
/**                                                     |                         |
 * @brief UART console init function                    |                         |
 */                                                     |                         |
static void Console_UART_Init(void)   <-----------------+                         |
{                                                                                 |
    console_uart.Instance = USART1;                                               |
    console_uart.Init.BaudRate = 115200;                                          |
    console_uart.Init.WordLength = UART_WORDLENGTH_8B;                            |
    console_uart.Init.StopBits = UART_STOPBITS_1;                                 |
    console_uart.Init.Parity = UART_PARITY_NONE;                                  |
    console_uart.Init.Mode = UART_MODE_TX_RX;                                     |
    console_uart.Init.HwFlowCtl = UART_HWCONTROL_NONE;                            |
    console_uart.Init.OverSampling = UART_OVERSAMPLING_16;                        |
    console_uart.Init.OneBitSampling = UART_ONE_BIT_SAMPLE_DISABLE;               |
    console_uart.AdvancedInit.AdvFeatureInit = UART_ADVFEATURE_NO_INIT;           |
    /**                                                                           |
     * typedef enum                                                               |
     * {                                                                          |
     *   COM1 = 0,                                                                |
     *   COM2 = 0,                                                                |
     * }COM_TypeDef;                                                              |
     */                                                                           |
    BSP_COM_Init(COM1,&console_uart);       <-------------------------------------+
}

#if (defined(__GNUC__) && !defined(__CC_ARM))
/* With GCC/RAISONANCE, small printf (option LD Linker->Libraries->Small printf
   set to 'Yes') calls __io_putchar() */
#define PUTCHAR_PROTOTYPE int __io_putchar(int ch)          --------------+   code/syscalls.c
#define GETCHAR_PROTOTYPE int __io_getchar(void)            --------------*--------------------+
#else                                                                     |                    |
#define PUTCHAR_PROTOTYPE int fputc(int ch, FILE *f)        --------------+                    |
#define GETCHAR_PROTOTYPE int fgetc(FILE *f)                --------------*--------------------+
#endif /* __GNUC__ */                                                     |                    |
                                                                          |                    |
/**                                                                       |                    |
 * @brief  Retargets the C library printf function to the USART.          |                    |
 * @param  None                                                           |                    |
 * @retval None                                                           |                    |
 */                                                                       |                    |
PUTCHAR_PROTOTYPE         <-----------------------------------------------+                    |
{                                                                                              |
    /* Place your implementation of fputc here */                                              |
    /* e.g. write a character to the USART2 and Loop until the end of transmission */          |
    while (HAL_OK != HAL_UART_Transmit(&console_uart, (uint8_t *) &ch, 1, 30000))              |
    {                                                                                          |
        ;                                                                                      |
    }                                                                                          |
    return ch;                                                                                 |
}                                                                                              |
                                                                                               |
/**                                                                                            |
 * @brief  Retargets the C library scanf function to the USART.                                |
 * @param  None                                                                                |
 * @retval None                                                                                |
 */                                                                                            |
GETCHAR_PROTOTYPE        <---------------------------------------------------------------------+
{
    /* Place your implementation of fgetc here */
    /* e.g. readwrite a character to the USART2 and Loop until the end of transmission */
    uint8_t ch = 0;
    while (HAL_OK != HAL_UART_Receive(&console_uart, (uint8_t *)&ch, 1, 30000))
    {
        ;
    }
    return ch;
}
```
