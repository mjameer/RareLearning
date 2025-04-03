### How to tell Maven to disregard SSL errors (and trusting all certs)?

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <profiles>
    <profile>
      <id>definedInM2SettingsXML</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
        <maven.wagon.http.ssl.insecure>true</maven.wagon.http.ssl.insecure>
        <maven.wagon.http.ssl.allowall>true</maven.wagon.http.ssl.allowall>
        <maven.wagon.http.ssl.ignore.validity.dates>true</maven.wagon.http.ssl.ignore.validity.dates>
      </properties>
    </profile>
  </profiles>
</settings>
```


### For IntelliJ

```
-Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true
```


<img width="973" alt="Screenshot 2024-11-08 at 9 54 38 PM" src="https://github.com/user-attachments/assets/dc35049b-bb51-4319-b9b7-927dab56668c">

<img width="970" alt="Screenshot 2024-11-08 at 10 02 20 PM" src="https://github.com/user-attachments/assets/39ab385e-4a85-451c-acc8-1a5a3c911250">



