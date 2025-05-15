# Continuous message monitoring in RDMA servers

The key to continuous RDMA monitoring is replacing your disconnect-and-cleanup approach with a persistent event loop that polls for new writes. The most reliable implementation uses RDMA Write with Immediate Data operations, which generate completion events on the server side whenever the client writes to your buffer.

## Core approach: Completion Queue polling

Instead of waiting for disconnect events, your server should continuously poll the Completion Queue (CQ) for new work completions:

```c
void continuous_monitoring(struct rdma_cm_id *id, struct ibv_mr *server_buffer_mr) {
    struct ibv_wc wc;
    struct ibv_cq *cq = id->qp->recv_cq;
    int ret;
    
    // Post initial receive buffer for RDMA write with immediate data
    post_receive_buffer(id->qp);
    
    printf("Server entering continuous monitoring mode...\n");
    
    while (1) {
        // Poll for completions
        ret = ibv_poll_cq(cq, 1, &wc);
        
        if (ret < 0) {
            fprintf(stderr, "Poll CQ failed\n");
            break;
        } else if (ret > 0) {
            if (wc.status != IBV_WC_SUCCESS) {
                fprintf(stderr, "Work completion error: %s\n", 
                        ibv_wc_status_str(wc.status));
                break;
            }
            
            if (wc.opcode == IBV_WC_RECV_RDMA_WITH_IMM) {
                // Client performed RDMA write with immediate
                uint32_t length = ntohl(wc.imm_data);
                
                // Process and print the data in ASCII
                printf("Received new data, length: %u bytes\n", length);
                print_buffer_ascii(server_buffer_mr->addr, length);
                
                // Post a new receive buffer for the next message
                post_receive_buffer(id->qp);
            }
        }
    }
}
```

## Client modifications required

For this approach to work, your client must use `IBV_WR_RDMA_WRITE_WITH_IMM` instead of regular RDMA writes:

```c
// Client code
struct ibv_send_wr wr;
// Initialize wr...
wr.opcode = IBV_WR_RDMA_WRITE_WITH_IMM;
wr.imm_data = htonl(data_length); // Pass message length as immediate data
wr.wr.rdma.remote_addr = remote_addr;
wr.wr.rdma.rkey = remote_rkey;
ibv_post_send(qp, &wr, &bad_wr);
```

The immediate data field (typically used to send the message length) generates a completion event that your server can detect.

## Displaying buffer contents in ASCII

To print buffer contents as ASCII when new data arrives:

```c
void print_buffer_ascii(void *buffer, size_t length) {
    unsigned char *data = (unsigned char*)buffer;
    size_t i;
    
    printf("Buffer content (ASCII): ");
    for (i = 0; i < length; i++) {
        // Print printable ASCII characters directly
        if (data[i] >= 32 && data[i] <= 126) {
            printf("%c", data[i]);
        } else {
            // Print non-printable characters as hex
            printf("\\x%02x", data[i]);
        }
    }
    printf("\n");
}
```

## Required receive buffer management

For RDMA Write with Immediate to work, you must post receive work requests to capture the immediate data:

```c
void post_receive_buffer(struct ibv_qp *qp) {
    struct ibv_recv_wr recv_wr = {};
    struct ibv_recv_wr *bad_recv_wr;
    struct ibv_sge sge = {};
    
    // No actual buffer needed for RDMA write with immediate
    // We're just posting to receive the immediate data
    
    // Post the receive work request
    if (ibv_post_recv(qp, &recv_wr, &bad_recv_wr)) {
        fprintf(stderr, "Failed to post receive buffer\n");
    }
}
```

## Alternative: Event-driven approach

If you want to avoid continuous CPU usage from polling, you can use the event-driven approach:

```c
void event_driven_monitoring(struct rdma_cm_id *id, struct ibv_mr *server_buffer_mr) {
    struct ibv_cq *ev_cq;
    void *ev_ctx;
    struct ibv_cq *cq = id->qp->recv_cq;
    struct ibv_comp_channel *comp_channel = id->send_cq_channel;
    
    // Request notification for the next completion event
    if (ibv_req_notify_cq(cq, 0)) {
        // Error handling
        return;
    }
    
    while (1) {
        // Wait for completion event (blocks until event occurs)
        if (ibv_get_cq_event(comp_channel, &ev_cq, &ev_ctx)) {
            // Error handling
            break;
        }
        
        // Acknowledge the event
        ibv_ack_cq_events(ev_cq, 1);
        
        // Request notification for the next event
        if (ibv_req_notify_cq(ev_cq, 0)) {
            // Error handling
            break;
        }
        
        // Process all available completions
        struct ibv_wc wc;
        while (ibv_poll_cq(cq, 1, &wc) > 0) {
            if (wc.status != IBV_WC_SUCCESS) {
                // Error handling
                fprintf(stderr, "Work completion error: %s\n", 
                        ibv_wc_status_str(wc.status));
                continue;
            }
            
            if (wc.opcode == IBV_WC_RECV_RDMA_WITH_IMM) {
                uint32_t length = ntohl(wc.imm_data);
                print_buffer_ascii(server_buffer_mr->addr, length);
                post_receive_buffer(id->qp);
            }
        }
    }
}
```

This blocks until events occur rather than constantly polling, saving CPU resources.

## Memory-polling fallback (not recommended)

If modifying the client to use RDMA Write with Immediate isn't possible, a less reliable approach is to poll memory locations for changes:

```c
void memory_polling_monitoring(struct ibv_mr *server_buffer_mr, size_t buffer_size) {
    // Header contains a marker and message length
    struct buffer_header {
        uint32_t marker;     // Magic value to detect new messages
        uint32_t length;     // Length of the message
    } *header = server_buffer_mr->addr;
    
    // Magic value that indicates new data
    uint32_t MAGIC_VALUE = 0xDEADBEEF;
    
    while (1) {
        // Check if new data has arrived
        if (header->marker == MAGIC_VALUE) {
            uint32_t length = header->length;
            
            // Print the data
            print_buffer_ascii((char*)server_buffer_mr->addr + sizeof(struct buffer_header), 
                               length);
            
            // Reset the marker to indicate we've processed the data
            header->marker = 0;
        }
        
        // Sleep to avoid excessive CPU usage
        usleep(1000); // 1ms
    }
}
```

With this approach, the client must write a marker and length before the actual data.

## Best practices

1. **Always repost receive buffers** after processing a message to keep the flow going
2. **Include error handling** for all RDMA operations  
3. **Consider thread safety** if your server handles multiple clients
4. **Implement flow control** to prevent buffer overruns with fast clients
5. **Add a graceful shutdown** mechanism to exit the monitoring loop

This continuous monitoring approach provides a more persistent RDMA server that can handle multiple messages from clients without disconnecting, while giving you immediate visibility into the buffer contents as ASCII whenever new data arrives.
