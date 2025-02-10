# Networking Fundamentals - Part 1

### 1. IP Address
- Unique identifier for devices in a network.
- **IPv4** consists of four bytes (32 bits) represented as `x.x.x.x` (0-255 per section).
- **Binary representation** explains why each section ranges from 0-255.

<img width="736" alt="image" src="https://github.com/user-attachments/assets/7ce4a918-09b6-4a0c-b0a6-35c2afbe9e42" />


### 2. Subnet & CIDR
- **Subnetting** divides a network into smaller segments for **security, isolation, and privacy**.

<img width="892" alt="image" src="https://github.com/user-attachments/assets/1d9bf5c4-11d3-4882-bcf3-8bb2e2c33884" />

- **CIDR (Classless Inter-Domain Routing)** defines the subnet size (e.g., `/24` means 256 addresses).
- **Public vs. Private Subnet:**
  - **Public** → Internet-accessible.
  - **Private** → No direct internet access.
 

### 3. Ports
- Ports distinguish different applications on a machine (e.g., `192.168.1.1:8080`).
- Some ports are reserved (e.g., MySQL: `3306`, Jenkins: `8080`).

## Next Topic (Part 2): OSI Model
- Covers **HTTP, TCP**, and **layers 1-7** in detail.

## Assignments
Try solving these CIDR-related questions:
1. What is the number of IP addresses for `172.68.3.0/30`?
2. What is the CIDR notation for `10.0.0.0/8`?

