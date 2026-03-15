# Direct Memory Access via High Performance AXI Slave Port

## Summary

This guide provides a practical walkthrough for implementing **Direct Memory Access (DMA)** on the *Zynq-7000 SoC* using the **AXI High-Performance (HP) Slave** ports. It focuses on the critical, often misunderstood relationship between the *DMA* engine and the *ARM Cortex-A9 CPU* cache. You will learn how to configure the *AXI DMA* in **polling mode**, perform data transfers between *DDR* memory and custom logic, and manually manage cache coherency to prevent common data synchronization failures.

## Table of Content

- Introduction
- Materials
- Methodology
    - Hardware
        - DMA Polling Architecture
    - Software
        - Cache Coherency
        - Simple Poll DMA
- Results
    - DMA Polling Implementation
    - Cache Coherency and Invalidation

## Introduction

In high-performance embedded systems, moving data efficiently is as important as the processing itself. In the *Zynq-7000* architecture, the *AXI High-Performance (HP) Slave* ports provide a dedicated, low-latency path between the *Programmable Logic (PL)* and the *DDR* controller.

## Materials

1. [Cora Z7-07S (Zynq-7000)](https://digilent.com/shop/cora-z7-zynq-7000-single-core-for-arm-fpga-soc-development/)
2. [Xilinx Design Pack 2018.3 (Vivado+SDK)](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/archive.html)
3. [Tera Term](https://github.com/TeraTermProject/teraterm/releases) for serial monitoring
4. [HxD](https://mh-nexus.de/en/downloads.php?product=HxD20) for creating binary files

## Methodology

### Hardware

**1. The Processor & Memory (Zynq-7000)**

The *Zynq PS* (Processing System) runs the software stack and manages the *DDR* Controller. It initializes the *DMA* transfers and allocates the memory buffers in the *DDR* that the hardware will use.

**2. The Interconnect (AXI SmartConnect)**

The *AXI SmartConnect* acts as the **bridge** between the *DMA’s Master* interfaces and the *PS High-Performance (HP) Slave* ports. **It is optimized for high-bandwidth data transfers**, ensuring the *DMA* can read from and write to the *DDR* with minimal latency.

**3. The Controller (AXI DMA)**

The *AXI DMA IP* acts as the **mover**. It converts *AXI4 Memory Mapped* transactions (from the *DDR*) into *AXI4-Stream* data (for the *FIFO*) and vice-versa. It offloads the data movement task from the *CPU*, allowing the processor to perform other tasks while data moves in the background.

**4. The Buffer/IP (AXI4-Stream Data FIFO)**

The *FIFO* provides clock domain crossing (if needed) and buffering to prevent data loss. In a real-world application, **this is where your Custom IP (like an FFT, filter, or encryption engine) would sit, processing the stream "on the fly**.

**5. The Data Path (Loopback)**

- **MM2S (Memory Map to Stream):** *DMA* reads data from *DDR* and sends it to the *FIFO*.

- **S2MM (Stream to Memory Map):** *DMA* receives data back from the *FIFO* and writes it back to a different location in the *DDR*.

#### DMA Polling Architecture
![](/docs/imgs/dma_hp_simple_poll_arch.png)

### Software

#### Cache Coherency

When using the *HP* ports, the *DMA* controller operates in a non-coherent domain. This means the *DMA* reads from and writes to the physical *DDR* memory without "snooping" the *CPU's L1/L2* caches.

If your software relies on the *CPU* to read or verify data written by the *DMA*, you will likely encounter synchronization errors. This happens because the *CPU* may serve a request from its cache (containing stale data) rather than the physical memory updated by the *DMA*.

While the *Accelerator Coherency Port (ACP)* exists to handle this automatically, the *HP* ports are the industry standard for high-bandwidth data movement—provided you take manual control of the cache. This article explains how to master that process using *Xil_DCache* functions to ensure your data pipeline remains consistent and reliable.

**1. Xil_DCacheFlushRange**

```c
Xil_DCacheFlushRange(address, length);
```
This pushes data from the Cache to the DDR. You use this before a DMA "Read" (moving data from RAM to a Peripheral). If the CPU wrote data to a variable, it might only exist in the cache. Flushing ensures the DDR actually has the latest values before the DMA controller looks there.

**2. Xil_DCacheInvalidateRange**

```c
Xil_DCacheInvalidateRange(address, length);
```
This tells the CPU, "The data you have in your cache is garbage; throw it away." This forces the CPU to refetch from the DDR. You use this after a DMA "Write" (Peripheral to RAM) is finished so the CPU sees the new data the DMA just dropped off.

#### Simple Poll DMA

**1. Configure the hardware**

```c
    XAxiDma_Config *CfgPtr;
    CfgPtr = XAxiDma_LookupConfig(DeviceId);
    if (!CfgPtr) {
        xil_printf("No config found for %d\r\n", DeviceId);
        return XST_FAILURE;
    }
    Status = XAxiDma_CfgInitialize(&AxiDma, CfgPtr);
    if (Status != XST_SUCCESS) {
        xil_printf("Initialization failed %d\r\n", Status);
        return XST_FAILURE;
    }
```

**2. Disable interrupts for polling mode**

```c
    // s2mm_intr: Device to DMA (Memory) Interrupt
    XAxiDma_IntrDisable(&AxiDma, XAXIDMA_IRQ_ALL_MASK,
                        XAXIDMA_DEVICE_TO_DMA);

    // mm2s_intr: DMA (Memory) to Device Interrupt
    XAxiDma_IntrDisable(&AxiDma, XAXIDMA_IRQ_ALL_MASK,
                        XAXIDMA_DMA_TO_DEVICE);
```

**3. Initiate the transfer**

```c
    // S2MM: Device to DMA (Memory)
    Status = XAxiDma_SimpleTransfer(&AxiDma,(UINTPTR) RxBufferPtr,
                                    MAX_PKT_LEN, XAXIDMA_DEVICE_TO_DMA);
    
    // MM2S: DMA (Memory) to Device
    Status = XAxiDma_SimpleTransfer(&AxiDma,(UINTPTR) TxBufferPtr,
                                    MAX_PKT_LEN, XAXIDMA_DMA_TO_DEVICE);
```

**Note: In Polling mode, we usually start the Receive (S2MM) before the Transmit (MM2S) to ensure the DMA is ready to catch the data as soon as the FIFO starts pushing it.**

**4. Wait for the transfer to finish**

```c
    while ((XAxiDma_Busy(&AxiDma,XAXIDMA_DEVICE_TO_DMA)) ||
        (XAxiDma_Busy(&AxiDma,XAXIDMA_DMA_TO_DEVICE))) {
    }
```

## Results

### DMA Polling Implementation

To verify the data flow, a breakpoint was inserted into `xaidma_example_simple_poll.c` at the completion of the **Transmit (TX) buffer initialization (line 272)**. This allows for manual interference with the memory before the *DMA* transfer begins.

- **Data Injection via JTAG**

With the processor halted, the *XSCT Console* was used to download a raw binary data file directly into the source memory address (*0x01100000*).

```bash
xsct% dow -data simple_poll_in_data.bin 0x01100000
```
**Note: When you use dow to load an ELF or binary, the debugger manages the cache for the memory regions it just touched. It knows it just wrote to those addresses, so it ensures the CPU sees the new binary code.**

Verification via the **Memory Monitor** confirmed that the *DDR* memory starting at *0x01100000* was successfully populated with the contents of `simple_poll_in_data.bin`.

![](/docs/imgs/dow_tx_mm_ss.png)

- **Execution and Verification**

Upon resuming execution, the *DMA* engine transferred the data from the source address to the destination address (*0x01300000*).

Verification via the **Memory Monitor** of the *DDR* memory starting at *0x01300000* matches the expected output, confirming a successful *DMA* transfer 

![](/docs/imgs/dow_rx_mm_ss.png)

The *UART* console output confirms a successful match between the sent and received data:
```
--- Entering main() ---
Data valid 0: sent 1 = received 1 PASS
Data valid 1: sent 2 = received 2 PASS
Data valid 2: sent 3 = received 3 PASS
Data valid 3: sent 4 = received 4 PASS
Data valid 4: sent 5 = received 5 PASS
Data valid 5: sent 6 = received 6 PASS
Data valid 6: sent 7 = received 7 PASS
Data valid 7: sent 8 = received 8 PASS
Data valid 8: sent 9 = received 9 PASS
Data valid 9: sent A = received A PASS
Data valid 10: sent B = received B PASS
Data valid 11: sent C = received C PASS
Data valid 12: sent D = received D PASS
Data valid 13: sent E = received E PASS
Data valid 14: sent F = received F PASS
Data valid 15: sent 10 = received 10 PASS
Data valid 16: sent 11 = received 11 PASS
Data valid 17: sent 12 = received 12 PASS
Data valid 18: sent 13 = received 13 PASS
Data valid 19: sent 14 = received 14 PASS
Data valid 20: sent 15 = received 15 PASS
Data valid 21: sent 16 = received 16 PASS
Data valid 22: sent 17 = received 17 PASS
Data valid 23: sent 18 = received 18 PASS
Data valid 24: sent 19 = received 19 PASS
Data valid 25: sent 1A = received 1A PASS
Data valid 26: sent 1B = received 1B PASS
Data valid 27: sent 1C = received 1C PASS
Data valid 28: sent 1D = received 1D PASS
Data valid 29: sent 1E = received 1E PASS
Data valid 30: sent 1F = received 1F PASS
Data valid 31: sent 20 = received 20 PASS
Successfully ran XAxiDma_SimplePoll Example
--- Exiting main() ---
```

To extract the results for external analysis, the **mrd** (*Memory Read*) command was used to save the destination memory back to a local file:

```bash
mrd -size b -bin -file simple_poll_out_data.bin {0x01300000} 0x20
```

Verification via the **HxD** of the `simple_poll_out_data.bin` file matches the expected output, confirming a successful *DDR* data extraction.

![](/docs/imgs/mrd_rx_file_ss.png)

### Cache Coherency and Invalidation

A critical aspect of *DMA* operations is managing the CPU Cache. To demonstrate the impact of cache coherency, the *Xil_DCacheInvalidateRange()* function was commented out within the data verification routine.

- **Observation**

When the cache invalidation is skipped, the system reports a *FAIL*, even if the data in the physical *DDR* is correct:

```
--- Entering main() ---
Data error 0: sent 1 != received C FAIL
XAxiDma_SimplePoll Example Failed
```

- **Technical Analysis**

This failure occurs because of how the *CPU* interacts with memory:

- **DMA Write:** The *AXI DMA* writes the received data directly into the *Physical DDR*, bypassing the *CPU's Cache*.

- **CPU Read:** When the *CPU* attempts to verify the data, it reads from its *L1/L2 Cache*. If the cache contains "stale" data from a previous operation, the *CPU* remains unaware that the underlying *DDR* has been updated by the DMA.

- **The Fix:** The Invalidate operation marks the cache lines as invalid, forcing the *CPU* to refetch the updated data directly from the *DDR*.