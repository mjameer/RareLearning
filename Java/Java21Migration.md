### Learning : 

If project uses FstCodec for Sterilization in Redis as it aids in Fast Serialization. Here is the summary of all the supported Serialization in redis

![image](https://github.com/user-attachments/assets/405f5b6a-1681-44f3-9661-68cd063acb51)

#### Issue : 

While migrating from Java 8 to Java 17, getting the following exception 

BeanInstantiationException: Failed to instantiate [org.redisson.api.RedissonClient]: Factory method 'redissonClient' threw exception; nested exception is com.fasterxml.jackson.databind.exc.ValueInstantiationException: Cannot construct instance of org.redisson.codec.FstCodec, problem: Unable to make field private final java.math.BigInteger java.math.BigDecimal.intVal accessible: module java.base does not "opens java.math"
 

#### Root cause : 

Java 17's stricter module system is preventing Jackson from accessing private fields in classes like BigDecimal, which is part of the java.base module.
 
#### Solution : 

Part 1: Add JVM Argument to Open the Module:{}

```xml
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.math=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/java.util.concurrent=ALL-UNNAMED
--add-opens java.base/java.nio=ALL-UNNAMED
--add-opens java.base/java.net=ALL-UNNAMED
--add-opens java.base/java.text=ALL-UNNAMED
--add-opens java.sql/java.sql=ALL-UNNAMED
```

Part 2: Ensure Redisson and Jackson Dependencies Are Up to Date

 Updated redisson from 3.11.6 to the latest. I may upgrade it as new version comes up

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.38.1</version>
</dependency>
```

Part 3: NoClassDefFoundError while trying to use FstCodec

https://github.com/redisson/redisson/issues/736 

```xml
  <dependency>
      <groupId>de.ruedigermoeller</groupId>
      <artifactId>fst</artifactId>
      <version>3.0.4-jdk17</version>
      <exclusions>
          <exclusion>
              <groupId>commons-collections</groupId>
              <artifactId>commons-collections</artifactId>
          </exclusion>
      </exclusions>
  </dependency>
 ```

 

Part 4: Update Model Mapper version : 

From 
```xml
<modelmapper.version>2.3.5</modelmapper.version>
```
to 
```xml
<modelmapper.version>3.2.1</modelmapper.version>
```
 
Note 2.3.5 is of Java 6. Extensive testing needed. 
