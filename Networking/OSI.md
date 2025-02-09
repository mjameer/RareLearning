# OSI Model Overview

The OSI (Open Systems Interconnection) model standardizes communication in computer systems and is used by devices like phones, routers, and computers. The video explains the OSI model from a software engineering perspective rather than a low-level networking view.

## The OSI Model Layers

### 1. Application Layer (Layer 7)
- Deals with user-facing applications like browsers.
- When a user requests a webpage (e.g., `http://10003:80`), an HTTP **GET** request is prepared with headers, cookies, and other metadata.

### 2. Presentation Layer (Layer 6)
- Handles encryption if necessary.
- In HTTP, this layer is bypassed, but in HTTPS, it encrypts data.

### 3. Session Layer (Layer 5)
- Manages sessions, ensuring that multiple communications between the same client and server are tracked properly.

### 4. Transport Layer (Layer 4)
- Splits data into segments and attaches **port numbers** (source and destination).
- Ensures that packets arrive in order.

### 5. Network Layer (Layer 3)
- Adds **IP addresses** (source and destination) and forms **packets**.
- Responsible for routing data.

### 6. Data Link Layer (Layer 2)
- Breaks packets into **frames**, assigns **MAC addresses** of sender and reciever, and detects basic errors. Its done by ARP Protocol (Address resolution protocol)

### 7. Physical Layer (Layer 1)
- Converts data into electrical signals, radio waves (Wi-Fi), or light signals (fiber optics) for transmission.

<img width="920" alt="image" src="https://github.com/user-attachments/assets/dd09afea-e625-441b-b0a9-214b5bac1163" />

## Data Transmission Process
1. A web request is initiated from a browser.
2. Data is processed through each OSI layer, adding headers and transforming it into smaller units.
3. The physical layer transmits data as electrical or radio signals.
4. The receiving device reconstructs the data by reversing the process.

## Security Considerations
- If unencrypted (e.g., HTTP), data can be intercepted on public networks.
- Encryption (e.g., HTTPS, VPNs) ensures secure data transmission.

### Reference

- [The OSI Model](https://www.youtube.com/watch?v=7IS7gigunyI)




