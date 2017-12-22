# libmodbus modbus_new_tcp_pi hacking

```C
modbus_t* modbus_new_tcp_pi(const char *node, const char *service)
{
    modbus_t *ctx;
    modbus_tcp_pi_t *ctx_tcp_pi;
    size_t dest_size;
    size_t ret_size;

    ctx = (modbus_t *)malloc(sizeof(modbus_t));
    if (ctx == NULL) {
        return NULL;
    }
    _modbus_init_common(ctx);

    /* Could be changed after to reach a remote serial Modbus device */
    // #define MODBUS_TCP_SLAVE         0xFF
    ctx->slave = MODBUS_TCP_SLAVE;

    /**
     * const modbus_backend_t _modbus_tcp_pi_backend = {
     *     _MODBUS_BACKEND_TYPE_TCP,
     *     _MODBUS_TCP_HEADER_LENGTH,
     *     _MODBUS_TCP_CHECKSUM_LENGTH,
     *     MODBUS_TCP_MAX_ADU_LENGTH,
     *     _modbus_set_slave,
     *     _modbus_tcp_build_request_basis,
     *     _modbus_tcp_build_response_basis,
     *     _modbus_tcp_prepare_response_tid,
     *     _modbus_tcp_send_msg_pre,
     *     _modbus_tcp_send,
     *     _modbus_tcp_receive,
     *     _modbus_tcp_recv,
     *     _modbus_tcp_check_integrity,
     *     _modbus_tcp_pre_check_confirmation,
     *     _modbus_tcp_pi_connect,
     *     _modbus_tcp_close,
     *     _modbus_tcp_flush,
     *     _modbus_tcp_select,
     *     _modbus_tcp_free
     * };
     * 
     * typedef struct _modbus_backend {
     *     unsigned int backend_type;
     *     unsigned int header_length;
     *     unsigned int checksum_length;
     *     unsigned int max_adu_length;
     *     int (*set_slave) (modbus_t *ctx, int slave);
     *     int (*build_request_basis) (modbus_t *ctx, int function, int addr,
     *                                 int nb, uint8_t *req);
     *     int (*build_response_basis) (sft_t *sft, uint8_t *rsp);
     *     int (*prepare_response_tid) (const uint8_t *req, int *req_length);
     *     int (*send_msg_pre) (uint8_t *req, int req_length);
     *     ssize_t (*send) (modbus_t *ctx, const uint8_t *req, int req_length);
     *     int (*receive) (modbus_t *ctx, uint8_t *req);
     *     ssize_t (*recv) (modbus_t *ctx, uint8_t *rsp, int rsp_length);
     *     int (*check_integrity) (modbus_t *ctx, uint8_t *msg,
     *                             const int msg_length);
     *     int (*pre_check_confirmation) (modbus_t *ctx, const uint8_t *req,
     *                                    const uint8_t *rsp, int rsp_length);
     *     int (*connect) (modbus_t *ctx);
     *     void (*close) (modbus_t *ctx);
     *     int (*flush) (modbus_t *ctx);
     *     int (*select) (modbus_t *ctx, fd_set *rset, struct timeval *tv, int msg_length);
     *     void (*free) (modbus_t *ctx);
     * } modbus_backend_t;
     */
    ctx->backend = &_modbus_tcp_pi_backend;

    /**
     * typedef struct _modbus_tcp_pi {
     *     /* Transaction ID */
     *     uint16_t t_id;
     *     /* TCP port */
     *     int port;
     *     /* Node */
     *     char node[_MODBUS_TCP_PI_NODE_LENGTH];
     *     /* Service */
     *     char service[_MODBUS_TCP_PI_SERVICE_LENGTH];
     * } modbus_tcp_pi_t;
     */
    ctx->backend_data = (modbus_tcp_pi_t *)malloc(sizeof(modbus_tcp_pi_t));
    if (ctx->backend_data == NULL) {
        modbus_free(ctx);
        errno = ENOMEM;
        return NULL;
    }
    ctx_tcp_pi = (modbus_tcp_pi_t *)ctx->backend_data;

    if (node == NULL) {
        /* The node argument can be empty to indicate any hosts */
        ctx_tcp_pi->node[0] = 0;
    } else {
        dest_size = sizeof(char) * _MODBUS_TCP_PI_NODE_LENGTH;
        ret_size = strlcpy(ctx_tcp_pi->node, node, dest_size);
        if (ret_size == 0) {
            fprintf(stderr, "The node string is empty\n");
            modbus_free(ctx);
            errno = EINVAL;
            return NULL;
        }

        if (ret_size >= dest_size) {
            fprintf(stderr, "The node string has been truncated\n");
            modbus_free(ctx);
            errno = EINVAL;
            return NULL;
        }
    }

    if (service != NULL) {
        dest_size = sizeof(char) * _MODBUS_TCP_PI_SERVICE_LENGTH;
        ret_size = strlcpy(ctx_tcp_pi->service, service, dest_size);
    } else {
        /* Empty service is not allowed, error catched below. */
        ret_size = 0;
    }

    if (ret_size == 0) {
        fprintf(stderr, "The service string is empty\n");
        modbus_free(ctx);
        errno = EINVAL;
        return NULL;
    }

    if (ret_size >= dest_size) {
        fprintf(stderr, "The service string has been truncated\n");
        modbus_free(ctx);
        errno = EINVAL;
        return NULL;
    }

    ctx_tcp_pi->t_id = 0;

    return ctx;
}
```
