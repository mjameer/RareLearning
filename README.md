# test

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


### Powershell

```
# Define the remote server and command
$remoteServer = "serverName"
$command = 'cmd.exe /c "cd C:\Windows\system32 && CLCONTROL /SYSTEM STATUS"'
 
# Define the username and password
$username = "id"  
$password = "pwd"  
 
 
# Create a credential object
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential($username, $securePassword)
 
try {
    # Use Invoke-Command to run the command on the remote server
    $result = Invoke-Command -ComputerName $remoteServer -Credential $credential -ScriptBlock { 
        param ($cmd)
        # Use cmd.exe to run the command
        cmd.exe /c $cmd
    } -ArgumentList $command
    # Convert the output to a string for easier processing
    $resultString = $result -join "`n"
    # Use a regex to find the Exit Code
    if ($resultString -match 'Exit Code ===> (\d+)') {
        $exitCode = $matches[1]
        Write-Output "Exit Code: $exitCode"
    } else {
        Write-Output "Exit Code not found in the output."
    }
} catch {
    Write-Error "An error occurred: $_"
}
```



```
@echo off

powershell -ExecutionPolicy Bypass -File "C:\path\to\your\script.ps1" arg1 arg2
```

```
This can be recieved by powershell via $args[0], $args[1]
```
