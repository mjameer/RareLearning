### Java StackTrace exception handling

```
LOGGER.error("Exception occurred: {} at {}({}:{})", 
    e.getMessage(), 
    e.getStackTrace()[0].getMethodName(), 
    e.getStackTrace()[0].getFileName(), 
    e.getStackTrace()[0].getLineNumber());
```

```
LOGGER.error("Exception occurred : {}", e.fillInStackTrace().getMessage());
```
