# Redis on Flash

## Overview
Redis on Flash is a feature of Redis Enterprise that extends storage beyond RAM into SSDs, optimizing cost and performance for large datasets. It stores frequently accessed ("hot") data in RAM while moving less-used ("warm") data to flash storage, ensuring efficiency and cost-effectiveness.

<img width="882" alt="image" src="https://github.com/user-attachments/assets/7574b3ea-30bd-4303-9474-91d33feb731e" />

## Key Benefits
- **Cost Savings**: Reduces memory costs compared to RAM-only Redis, with savings up to 50%.
- **Optimized Performance**: Keeps hot data in RAM and migrates warm data to SSDs as needed.
- **Scalability**: Ideal for large datasets where the working set is significantly smaller than the full data set.
- **Faster than Traditional Databases**: Can replace slower data stores like MongoDB or DynamoDB while maintaining high throughput.
- **High Performance**: Benchmarks show over 3 million operations per second with sub-millisecond latency using NVMe SSDs.

## Setup Guide
### Enabling Redis on Flash
1. **During Node Setup**
   - Tick the **Enable Flash Storage Support** box.
   - Specify a path to the flash storage if applicable.

2. **Creating a Database**
   - Select **Flash** as the storage option.
   - Use the slider to specify the RAM limit for hot data.
   - Click **Activate** to complete the setup.

## Use Cases
- **Growing Datasets**: Ideal for data sets that outgrow available RAM.
- **Cost Optimization**: Reduces infrastructure costs while maintaining high performance.
- **Migrating from Slower Databases**: Possible to move from MongoDB or DynamoDB to Redis on Flash for better performance and lower costs.

## Performance Metrics
- Over **3 million operations per second** with **sub-millisecond latency**.
- **1GB/s data transfer rate** to and from disk on an Intel NVMe SSD.

