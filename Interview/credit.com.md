# Find Data in File A Not Present in File B Using Java

## ❓ Problem

You have two large files: **File A** and **File B**. You want to find the lines that are **present in File A but not in File B**, using Java.

---

## ✅ Strategy Based on File Size

### 💡 Assumptions
- Both files are **huge** (e.g., gigabytes).
- Each line represents a meaningful data entry (e.g., string, ID, JSON).
- Java 8 or above is used.

---

## 🔁 General Approach


### ✅ Do
Load the smaller of the two (preferably File B) into a `Set` and then stream File A line-by-line to check which lines are not present in File B.


```java
import java.io.IOException;
import java.nio.file.*;
import java.util.*;
import java.util.stream.*;

public class SmartFileDifferenceFinder {

    public static void main(String[] args) throws IOException {
        Path fileA = Paths.get("fileA.txt");
        Path fileB = Paths.get("fileB.txt");
        Path output = Paths.get("onlyInSmaller.txt");

        // Compare file sizes
        long sizeA = Files.size(fileA);
        long sizeB = Files.size(fileB);

        Path smallerFile = sizeA <= sizeB ? fileA : fileB;
        Path largerFile = sizeA <= sizeB ? fileB : fileA;

        // Load the smaller file into a Set
        Set<String> smallerSet = Files.lines(smallerFile)
                                      .collect(Collectors.toSet());

        // Stream the larger file line by line and write only those not in the smaller file
        try (BufferedWriter writer = Files.newBufferedWriter(output)) {
            Files.lines(largerFile)
                 .filter(line -> !smallerSet.contains(line))
                 .forEach(line -> {
                     try {
                         writer.write(line);
                         writer.newLine();
                     } catch (IOException e) {
                         throw new UncheckedIOException(e);
                     }
                 });
        }

        System.out.println("Done. Output written to: " + output.toString());
    }
}
```

### 📌 Benefit
This ensures that the file loaded into memory (as a `Set`) is the smaller one, reducing the risk of `OutOfMemoryError`.


## Disk Storage approach 

```
    <!--
        MapDB provides concurrent Maps, Sets and Queues backed by disk storage or off-heap memory.
        It is a fast, scalable and easy to use embedded Java database.
    -->
        <dependency>
            <groupId>org.mapdb</groupId>
            <artifactId>mapdb</artifactId>
            <version>3.1.0</version>
        </dependency>
```


```java
package com.demo;

import org.mapdb.*;
import java.io.*;
import java.nio.file.*;
import java.util.Set;

public class MapDBDiskBackedDiff {

    public static void main(String[] args) throws IOException {
        DB db = DBMaker.fileDB("fileB.db").fileMmapEnable().make();
        Set<String> diskSet = db.hashSet("fileBSet", Serializer.STRING).createOrOpen();

        // Load File B into disk-backed set
        Files.lines(Paths.get("fileB.txt")).forEach(diskSet::add);

        // Stream through File A
        try (BufferedWriter writer = Files.newBufferedWriter(Paths.get("onlyInA.txt"))) {
            Files.lines(Paths.get("fileA.txt"))
                    .filter(line -> !diskSet.contains(line))
                    .forEach(line -> {
                        try {
                            writer.write(line);
                            writer.newLine();
                        } catch (IOException e) {
                            throw new UncheckedIOException(e);
                        }
                    });
        }

        db.close();
    }
}

```



---

## 📌 When to Use MapDB

Use MapDB if:
- Your files are **too large** to fit into memory.
- You need an **embedded**, lightweight solution.
- You're working in a **non-distributed**, single JVM environment.
- You want **persisted**, resumable disk-backed data structures.
