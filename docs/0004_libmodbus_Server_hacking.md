# libmodbus Server hacking

[stephane/libmodbus](https://github.com/stephane/libmodbus)

```C
/*
 * Copyright © 2008-2014 Stéphane Raimbault <stephane.raimbault@gmail.com>
 *
 * SPDX-License-Identifier: BSD-3-Clause
 */

#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>
#include <modbus.h>
#ifdef _WIN32
# include <winsock2.h>
#else
# include <sys/socket.h>
#endif

/* For MinGW */
#ifndef MSG_NOSIGNAL
# define MSG_NOSIGNAL 0
#endif

#include "unit-test.h"

// 枚举三种类型
enum {
    TCP,        // 本地端的TCP
    TCP_PI,     // IP地址
    RTU         // UART/RS232/RS485/RS422
};

int main(int argc, char*argv[])
{
    int s = -1;
    modbus_t *ctx;
    modbus_mapping_t *mb_mapping;
    int rc;
    int i;
    int use_backend;
    uint8_t *query;
    int header_length;

    if (argc > 1) {
        if (strcmp(argv[1], "tcp") == 0) {              // 匹配tcp
            use_backend = TCP;
        } else if (strcmp(argv[1], "tcppi") == 0) {     // 匹配tcp/ip
            use_backend = TCP_PI;
        } else if (strcmp(argv[1], "rtu") == 0) {       // 匹配rtu
            use_backend = RTU;
        } else {
            printf("Usage:\n  %s [tcp|tcppi|rtu] - Modbus server for unit testing\n\n", argv[0]);
            return -1;
        }
    } else {
        /* By default */
        use_backend = TCP;
    }

    if (use_backend == TCP) {
        ctx = modbus_new_tcp("127.0.0.1", 1502);
        /**
         * Modbus_Application_Protocol_V1_1b.pdf Chapter 4 Section 1 Page 5
         * TCP MODBUS ADU = 253 bytes + MBAP (7 bytes) = 260 bytes
         * 
         * #define MODBUS_TCP_MAX_ADU_LENGTH  260
         */
        query = malloc(MODBUS_TCP_MAX_ADU_LENGTH);
    } else if (use_backend == TCP_PI) {
        ctx = modbus_new_tcp_pi("::0", "1502");
        query = malloc(MODBUS_TCP_MAX_ADU_LENGTH);
    } else {
        ctx = modbus_new_rtu("/dev/ttyUSB0", 115200, 'N', 8, 1);
        modbus_set_slave(ctx, SERVER_ID);
        query = malloc(MODBUS_RTU_MAX_ADU_LENGTH);
    }
    header_length = modbus_get_header_length(ctx);                      // 获取Modbus协议头长度

    modbus_set_debug(ctx, TRUE);                                        // 打开debug模式

    /**
     * allocate four arrays of bits and registers accessible from their starting addresses
     */
    mb_mapping = modbus_mapping_new_start_address(
        /**
         * const uint16_t UT_BITS_ADDRESS = 0x130;
         * const uint16_t UT_BITS_NB = 0x25;
         */
        UT_BITS_ADDRESS, UT_BITS_NB,
        /**
         * const uint16_t UT_INPUT_BITS_ADDRESS = 0x1C4;
         * const uint16_t UT_INPUT_BITS_NB = 0x16;
         */
        UT_INPUT_BITS_ADDRESS, UT_INPUT_BITS_NB,
        /**
         * const uint16_t UT_REGISTERS_ADDRESS = 0x160;
         * const uint16_t UT_REGISTERS_NB = 0x3;
         */
        UT_REGISTERS_ADDRESS, UT_REGISTERS_NB_MAX,
        /**
         * const uint16_t UT_INPUT_REGISTERS_ADDRESS = 0x108;
         * const uint16_t UT_INPUT_REGISTERS_NB = 0x1;
         */
        UT_INPUT_REGISTERS_ADDRESS, UT_INPUT_REGISTERS_NB);
    if (mb_mapping == NULL) {
        fprintf(stderr, "Failed to allocate the mapping: %s\n",
                modbus_strerror(errno));
        modbus_free(ctx);
        return -1;
    }

    /* Examples from PI_MODBUS_300.pdf.
       Only the read-only input values are assigned. */

    /* Initialize input values that's can be only done server side. */
    /**
     *  /* Sets many bits from a table of bytes (only the bits between idx and
     *     idx + nb_bits are set) */
     *  void modbus_set_bits_from_bytes(uint8_t *dest, int idx, unsigned int nb_bits,
     *                                  const uint8_t *tab_byte)
     *  {
     *      unsigned int i;
     *      int shift = 0;
     *
     *      for (i = idx; i < idx + nb_bits; i++) {                         // 从第idx开始copy
     *          dest[i] = tab_byte[(i - idx) / 8] & (1 << shift) ? 1 : 0;   // 从没一个Byte的bit中获取位来设置dest的值
     *          /* gcc doesn't like: shift = (++shift) % 8; */
     *          shift++;
     *          shift %= 8;
     *      }
     *  }
     *
     * const uint16_t UT_INPUT_BITS_NB = 0x16;
     * const uint8_t UT_INPUT_BITS_TAB[] = { 0xAC, 0xDB, 0x35 };
     */
    modbus_set_bits_from_bytes(mb_mapping->tab_input_bits, 0, UT_INPUT_BITS_NB,
                               UT_INPUT_BITS_TAB);

    /* Initialize values of INPUT REGISTERS */
    for (i=0; i < UT_INPUT_REGISTERS_NB; i++) {
        /** const uint16_t UT_INPUT_REGISTERS_TAB[] = { 0x000A }; */
        mb_mapping->tab_input_registers[i] = UT_INPUT_REGISTERS_TAB[i];;
    }

    // 监听并等待连接
    if (use_backend == TCP) {
        s = modbus_tcp_listen(ctx, 1);
        modbus_tcp_accept(ctx, &s);
    } else if (use_backend == TCP_PI) {
        s = modbus_tcp_pi_listen(ctx, 1);
        modbus_tcp_pi_accept(ctx, &s);
    } else {
        rc = modbus_connect(ctx);
        if (rc == -1) {
            fprintf(stderr, "Unable to connect %s\n", modbus_strerror(errno));
            modbus_free(ctx);
            return -1;
        }
    }

    for (;;) {
        do {
            /**
             * /* Receive the request from a modbus master */
             * int modbus_receive(modbus_t *ctx, uint8_t *req)
             * {
             *     if (ctx == NULL) {
             *         errno = EINVAL;
             *         return -1;
             *     }

             *     return ctx->backend->receive(ctx, req);
             * }
             */
            // 接收指示请求
            rc = modbus_receive(ctx, query);
            /* Filtered queries return 0 */
        } while (rc == 0);

        /* The connection is not closed on errors which require on reply such as
           bad CRC in RTU. */
        if (rc == -1 && errno != EMBBADCRC) {
            /* Quit */
            break;
        }

        /* Special server behavior to test client */
        // 这里一定要注意里面的continue的用法，后面的下列语句是不一定会运行到的
        // rc = modbus_reply(ctx, query, rc, mb_mapping);
        if (query[header_length] == 0x03) {                             // function code
            /* Read holding registers */

            // #define MODBUS_GET_INT16_FROM_INT8(tab_int8, index) ((tab_int8[(index)] << 8) + tab_int8[(index) + 1])
            if (MODBUS_GET_INT16_FROM_INT8(query, header_length + 3)
                /* const uint16_t UT_REGISTERS_NB_SPECIAL = 0x2; */
                == UT_REGISTERS_NB_SPECIAL) {
                printf("Set an incorrect number of values\n");
                MODBUS_SET_INT16_TO_INT8(query, header_length + 3,
                                         UT_REGISTERS_NB_SPECIAL - 1);
            } else if (MODBUS_GET_INT16_FROM_INT8(query, header_length + 1)
                       == UT_REGISTERS_ADDRESS_SPECIAL) {
                printf("Reply to this special register address by an exception\n");
                modbus_reply_exception(ctx, query,
                                       MODBUS_EXCEPTION_SLAVE_OR_SERVER_BUSY);
                continue;
            } else if (MODBUS_GET_INT16_FROM_INT8(query, header_length + 1)
                       == UT_REGISTERS_ADDRESS_INVALID_TID_OR_SLAVE) {
                const int RAW_REQ_LENGTH = 5;
                uint8_t raw_req[] = {
                    (use_backend == RTU) ? INVALID_SERVER_ID : 0xFF,
                    0x03,
                    0x02, 0x00, 0x00
                };

                printf("Reply with an invalid TID or slave\n");
                modbus_send_raw_request(ctx, raw_req, RAW_REQ_LENGTH * sizeof(uint8_t));
                continue;
            } else if (MODBUS_GET_INT16_FROM_INT8(query, header_length + 1)
                       == UT_REGISTERS_ADDRESS_SLEEP_500_MS) {
                printf("Sleep 0.5 s before replying\n");
                usleep(500000);
            } else if (MODBUS_GET_INT16_FROM_INT8(query, header_length + 1)
                       == UT_REGISTERS_ADDRESS_BYTE_SLEEP_5_MS) {
                /* Test low level only available in TCP mode */
                /* Catch the reply and send reply byte a byte */
                uint8_t req[] = "\x00\x1C\x00\x00\x00\x05\xFF\x03\x02\x00\x00";
                int req_length = 11;
                int w_s = modbus_get_socket(ctx);
                if (w_s == -1) {
                    fprintf(stderr, "Unable to get a valid socket in special test\n");
                    continue;
                }

                /* Copy TID */
                req[1] = query[1];
                for (i=0; i < req_length; i++) {
                    printf("(%.2X)", req[i]);
                    usleep(5000);
                    rc = send(w_s, (const char*)(req + i), 1, MSG_NOSIGNAL);
                    if (rc == -1) {
                        break;
                    }
                }
                continue;
            }
        }

        // 前面感觉都是小打小闹的，貌似这里才是重点需要分析的地方，下一小节专门分析
        rc = modbus_reply(ctx, query, rc, mb_mapping);
        if (rc == -1) {
            break;
        }
    }

    printf("Quit the loop: %s\n", modbus_strerror(errno));

    if (use_backend == TCP) {
        if (s != -1) {
            close(s);
        }
    }
    modbus_mapping_free(mb_mapping);
    free(query);
    /* For RTU */
    modbus_close(ctx);
    modbus_free(ctx);

    return 0;
}
```
