# Database Indexing Overview  

## Introduction  
This document provides a high-level overview of database indexing, including:  
- **What is an Index?**  
- **The Problem it Solves**  
- **Common Index Types and When to Use Them**  

## What is an Index?  
An index is a **data structure stored on disk** that acts as a **map** to quickly locate data in a database, reducing the need for full table scans.  

## The Problem Indexing Solves  
Without an index, finding a record requires **scanning pages sequentially**, which is slow.  
- **Example:** If a table has **100 million users**, and each page contains **100 rows**, then a worst-case search requires scanning **1 million pages**.
  
<img width="916" alt="image" src="https://github.com/user-attachments/assets/ad60e39c-bd22-4885-aba7-f4492265d54b" />






- **Solution:** An index tells the database **exactly where** to look, reducing the number of pages accessed.

  
![image](https://github.com/user-attachments/assets/588d95f0-bc54-4949-86dc-64165a29e47d)

---

## Types of Indexes  

### **1. B-Tree Index (Most Common) 🌳**  
- **Structure:** A balanced tree where nodes store sorted values and pointers to data pages.  
- **Best For:**  
  - **Exact lookups** (`WHERE age = 30`)  
  - **Range queries** (`WHERE age > 30 AND age < 50`)  
  - **Sorting** (`ORDER BY age`)  
- **Used In:** PostgreSQL, MySQL, Oracle, etc.

  
<img width="1166" alt="image" src="https://github.com/user-attachments/assets/da89e24d-a3af-490d-94c5-871031301108" />

### **2. Hash Index 🏷️**  
- **Structure:** Uses a **hash function** to map keys to data locations.  
- **Best For:** **O(1) exact lookups** (`WHERE email = "user@example.com"`)  
- **Limitations:**  
  - **No range queries or sorting**  
  - Rarely used in production databases  
- **Used In:** Redis, Memcached (in-memory databases)


<img width="896" alt="image" src="https://github.com/user-attachments/assets/5fdb119a-d752-4422-bc6a-ad02d1fbd182" />

---

## When B-Trees Are Not Ideal  


<img width="649" alt="image" src="https://github.com/user-attachments/assets/d5299974-6b52-4b32-b4dd-32872089fb0f" />

### **1. Geospatial Data (Latitude/Longitude Searches) 🌍**  
- **B-Trees struggle with 2D data** (e.g., **finding nearby locations**).  
- Instead, use **Geospatial Indexes**:  
  - **Geohashing** (Used in Redis, converts coordinates into short strings for fast lookups)  
  - **Quad Trees** (Adaptive splitting of dense regions)  
  - **R-Trees** (Clusters nearby data with overlapping bounding boxes, used in **PostGIS**)  

<img width="903" alt="image" src="https://github.com/user-attachments/assets/7b086f63-1656-43f9-b0a8-13ebc5575865" />

### **2. Full-Text Search (Finding Words in Text) 🔍**  
- **Problem:** B-Trees only help with **prefix searches** (e.g., `WHERE name LIKE 'pizza%'`).  
- **Solution:** Use **Inverted Indexes** (maps words to document locations).  
- **Used In:** Elasticsearch, PostgreSQL Full-Text Search, Lucene.
  
<img width="608" alt="image" src="https://github.com/user-attachments/assets/f142107e-931a-4719-8f41-ebb0985083c9" />

---

## Choosing the Right Index 🏆  

| **Use Case**          | **Recommended Index** |
|-----------------------|----------------------|
| **Text search** (`LIKE '%word%'`) | **Inverted Index** (Elasticsearch, PostgreSQL Full-Text) |
| **Geospatial data** (lat/lon) | **Geospatial Index** (Geohashing, R-Trees, Quad Trees) |
| **Fast exact lookups in memory** | **Hash Index** (Redis, Memcached) |
| **Everything else** | **B-Tree Index** (Default) |

<img width="731" alt="image" src="https://github.com/user-attachments/assets/b9ce75b5-fc0b-4c6c-84a4-d43ea5cf888f" />

---

## Final Takeaways 🎯  
- **Identify inefficient queries** before applying indexes.  
- **Choose the right column(s) for indexing** based on query patterns.  
- **Use specialized indexes (Geospatial, Full-Text, Hash) only when necessary.**  

Indexes **optimize data retrieval** but should be used **strategically** to avoid unnecessary overhead. 🚀  
