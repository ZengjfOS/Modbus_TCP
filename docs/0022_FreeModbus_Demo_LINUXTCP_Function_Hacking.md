# FreeModbus Demo LINUXTCP Function Hacking

这份demo中只支持2种Modbus功能，另外还有地址限制，具体请参考源代码。

## Source Code

[refers/freemodbus-v1.5.0.zip](refers/freemodbus-v1.5.0.zip)

## Docs

* [FreeModbus undefined reference to `pthread_create'](http://www.cnblogs.com/zengjfgit/p/8175723.html)
* [FreeModbus LINUXTCP Compile ERROR](http://www.cnblogs.com/zengjfgit/p/8175831.html)
* [mbpoll Test FreeModbus TCP Demo](http://www.cnblogs.com/zengjfgit/p/8176246.html)

## IP/Port

```C
#define MB_TCP_PORT_USE_DEFAULT 0

int
main( int argc, char *argv[] )
{
    ...
    if( eMBTCPInit( MB_TCP_PORT_USE_DEFAULT ) != MB_ENOERR )           -----------------+
    {                                                                                   |
        fprintf( stderr, "%s: can't initialize modbus stack!\r\n", PROG );              |
        iExitCode = EXIT_FAILURE;                                                       |
    }                                                                                   |
    ...                                                                                 |
}                                                                                       |
                                                                                        |
eMBErrorCode                                                                            |
eMBTCPInit( USHORT ucTCPPort )         <------------------------------------------------+
{
    eMBErrorCode    eStatus = MB_ENOERR;

    if( ( eStatus = eMBTCPDoInit( ucTCPPort ) ) != MB_ENOERR )      ------------+
    {                                                                           |
        eMBState = STATE_DISABLED;                                              |
    }                                                                           |
    ...                                                                         |
}                                                                               |
                                                                                |
eMBErrorCode                                                                    |
eMBTCPDoInit( USHORT ucTCPPort )         <--------------------------------------+
{
    eMBErrorCode    eStatus = MB_ENOERR;

    if( xMBTCPPortInit( ucTCPPort ) == FALSE )       ------------+
    {                                                            |
        eStatus = MB_EPORTERR;                                   |
    }                                                            |
    return eStatus;                                              |
}                                                                |
                                                                 |
BOOL                                                             |
xMBTCPPortInit( USHORT usTCPPort )          <--------------------+
{
    USHORT          usPort;
    struct sockaddr_in serveraddr;

    if( usTCPPort == 0 )
    {
        usPort = MB_TCP_DEFAULT_PORT;       ------------------------------+
    }                                                                     |
    else                                                                  |
    {                                                                     |
        usPort = ( USHORT ) usTCPPort;                                    |
    }                                                                     |
    ...                                                                   |
}                                                                         |
                                                                          |
#define MB_TCP_DEFAULT_PORT 502 /* TCP listening port. */      <----------+
```

## TCP Poll

```C
int
main( int argc, char *argv[] )
{
    ...
    case  'e' :
        if( bCreatePollingThread(  ) != TRUE )                 ---------------------------+
        {                                                                                 |
            printf(  "Can't start protocol stack! Already running?\r\n"  );               |
        }                                                                                 |
        break;                                                                            |
    ...                                                                                   |
}                                                                                         |
                                                                                          |
BOOL                                                                                      |
bCreatePollingThread( void )           <--------------------------------------------------+
{
    BOOL            bResult;
        pthread_t       xThread;
    if( eGetPollingThreadState(  ) == STOPPED )
    {
        if( pthread_create( &xThread, NULL, pvPollingThread, NULL ) != 0 )
        {                                             |
            /* Can't create the polling thread. */    |
            bResult = FALSE;                          |
        }                                             |
        else                                          |
        {                                             |
            bResult = TRUE;                           |
        }                                             |
    }                                                 |
    else                                              |
    {                                                 |
        bResult = FALSE;                              |
    }                                                 |
                                                      |
    return bResult;                                   |
}                                                     |
                                                      |
void* pvPollingThread( void *pvParameter )    <-------+
{
    eSetPollingThreadState( RUNNING );

    if( eMBEnable(  ) == MB_ENOERR )
    {
        do
        {
            if( eMBPoll(  ) != MB_ENOERR )
                break;
        }
        while( eGetPollingThreadState(  ) != SHUTDOWN );          <----------注意这里
    }

    ( void )eMBDisable(  );

    eSetPollingThreadState( STOPPED );

    return 0;
}
```

## Data Parser

四个处理函数。

```C
eMBErrorCode
eMBRegInputCB( UCHAR * pucRegBuffer, USHORT usAddress, USHORT usNRegs )
{
    eMBErrorCode    eStatus = MB_ENOERR;
    int             iRegIndex;

    printf("usAddress = %d, usNRegs = %d.\r\n.", usAddress, usNRegs);
    /**
     * #define REG_INPUT_START 1000
     * #define REG_INPUT_NREGS 4
     */
    if( ( usAddress >= REG_INPUT_START )
        && ( usAddress + usNRegs <= REG_INPUT_START + REG_INPUT_NREGS ) )
    {
        iRegIndex = ( int )( usAddress - usRegInputStart );
        while( usNRegs > 0 )
        {
            *pucRegBuffer++ = ( unsigned char )( usRegInputBuf[iRegIndex] >> 8 );
            *pucRegBuffer++ = ( unsigned char )( usRegInputBuf[iRegIndex] & 0xFF );
            iRegIndex++;
            usNRegs--;
        }
    }
    else
    {
        eStatus = MB_ENOREG;
    }

    return eStatus;
}

eMBErrorCode
eMBRegHoldingCB( UCHAR * pucRegBuffer, USHORT usAddress, USHORT usNRegs, eMBRegisterMode eMode )
{
    eMBErrorCode    eStatus = MB_ENOERR;
    int             iRegIndex;

    /**
     * #define REG_HOLDING_START 2000
     * #define REG_HOLDING_NREGS 130
     */
    if( ( usAddress >= REG_HOLDING_START ) &&
        ( usAddress + usNRegs <= REG_HOLDING_START + REG_HOLDING_NREGS ) )
    {
        iRegIndex = ( int )( usAddress - usRegHoldingStart );
        switch ( eMode )
        {
            /* Pass current register values to the protocol stack. */
        case MB_REG_READ:
            while( usNRegs > 0 )
            {
                *pucRegBuffer++ = ( UCHAR ) ( usRegHoldingBuf[iRegIndex] >> 8 );
                *pucRegBuffer++ = ( UCHAR ) ( usRegHoldingBuf[iRegIndex] & 0xFF );
                iRegIndex++;
                usNRegs--;
            }
            break;

            /* Update current register values with new values from the
             * protocol stack. */
        case MB_REG_WRITE:
            while( usNRegs > 0 )
            {
                usRegHoldingBuf[iRegIndex] = *pucRegBuffer++ << 8;
                usRegHoldingBuf[iRegIndex] |= *pucRegBuffer++;
                iRegIndex++;
                usNRegs--;
            }
        }
    }
    else
    {
        eStatus = MB_ENOREG;
    }
    return eStatus;
}

eMBErrorCode
eMBRegCoilsCB( UCHAR * pucRegBuffer, USHORT usAddress, USHORT usNCoils, eMBRegisterMode eMode )
{
    return MB_ENOREG;
}

eMBErrorCode
eMBRegDiscreteCB( UCHAR * pucRegBuffer, USHORT usAddress, USHORT usNDiscrete )
{
    return MB_ENOREG;
}
```
