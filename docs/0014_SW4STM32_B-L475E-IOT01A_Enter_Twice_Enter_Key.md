# SW4STM32 B-L475E-IOT01A Enter Twice Enter Key

如下Source Code中，在获取到了`\r`之后，会再读取一个字符，但这个字符不会保存，被认为是`\n`。

## Source Code

```C
/**
 * @brief  Get a line from the console (user input).
 * @param  Out:  inputString   Pointer to buffer for input line.
 * @param  In:   len           Max length for line.
 * @retval Number of bytes read from the terminal.
 */
int getInputString(char *inputString, size_t len)
{
    size_t currLen = 0;
    int c = 0;

    c = getchar();

    while ((c != EOF) && ((currLen + 1) < len) && (c != '\r') && (c != '\n') )
    {
        if (c == '\b')
        {
            if (currLen != 0)
            {
                --currLen;
                inputString[currLen] = 0;
                msg_info(" \b");
            }
        }
        else
        {
            if (currLen < (len-1))
            {
                inputString[currLen] = c;
            }

            ++currLen;
        }
        c = getchar();
    }
    if (currLen != 0)
    { /* Close the string in the input buffer... only if a string was written to it. */
        inputString[currLen] = '\0';
    }
    if (c == '\r')
    {
        c = getchar(); /* assume there is '\n' after '\r'. Just discard it. */
    }

    return currLen;
}
```

