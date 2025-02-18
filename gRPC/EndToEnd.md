# Protobuf

Protocol Buffers (Protobuf) is an efficient binary serialization format, reducing size and improving performance over JSON.

## Proto File

A proto file is platform- and language-neutral. However, the JVM is not platform-neutral and requires installation based on the OS. After creating a proto file, a tool is needed to compile it into Java source code. This is where the Protobuf compiler (protoc) comes in. Since protoc is OS-dependent, a plugin helps by automatically downloading the appropriate version for the operating system (e.g., Mac or Windows) and generating the necessary source code.

## Building a New Object

![image](https://github.com/user-attachments/assets/2838c71f-5a00-40bd-a4ed-c928f02e58f5)

## Updating an Existing Object

![image](https://github.com/user-attachments/assets/aaa80bf6-4980-4a2b-b1e8-ff5483540f61)

Note that objects created are immutable, so setters won't work.

## Proto and Null Values

![image](https://github.com/user-attachments/assets/a05d7b5f-d284-4e0e-85e3-eeb4876e89a0)

![image](https://github.com/user-attachments/assets/5795c260-8df2-4e07-8337-9bcd144efd2c)

If you want to remove a name, use the `clearName` method.

![image](https://github.com/user-attachments/assets/1765cd20-d125-43d6-9fcb-23608a89b9e6)

## Scalar Types

![image](https://github.com/user-attachments/assets/b7e3fdcc-dc4e-411e-ad3b-d9a428f4530c)

## Compositions

<img width="643" alt="image" src="https://github.com/user-attachments/assets/957d83bb-14b0-4ff4-9cac-c7a88e4f1f34" />

## Collections

<img width="742" alt="image" src="https://github.com/user-attachments/assets/04b6b7fd-9dac-4944-a6df-c30b2f8875d5" />

### List or Set

![image](https://github.com/user-attachments/assets/55a5158f-d9e4-463d-93bd-181d1c102158)

![image](https://github.com/user-attachments/assets/9a521786-ce9e-4d61-b01b-ab583bb71524)

**Note:** For sets, it is your responsibility to enforce uniqueness; Proto does not enforce it.

### Map

![image](https://github.com/user-attachments/assets/f2f820dd-0b0f-43a8-ba19-95c1ef43805b)

![image](https://github.com/user-attachments/assets/1a4d242b-725e-4bb8-8b7b-d2967266780e)

![image](https://github.com/user-attachments/assets/5f32954d-5df8-4b49-bbdc-820b3610c75c)

### Complex Data Structures

![image](https://github.com/user-attachments/assets/99268fcd-d765-4593-962d-169dbfc0576c)

## Enums

<img width="480" alt="image" src="https://github.com/user-attachments/assets/659ddc74-4da6-431e-98d0-d3afc2ced345" />


![image](https://github.com/user-attachments/assets/e99dea10-ddd4-4f5d-9496-79c07db0f7a0)

**Note:** Enums always need to start with `0` as a default value.

### Default Values for Enums

<img width="753" alt="image" src="https://github.com/user-attachments/assets/31e1cfee-4d47-43dd-8aa6-fe6660eb520c" />

## OneOf

<img width="548" alt="image" src="https://github.com/user-attachments/assets/6ecd4aa9-0ba6-4247-bfcd-2512ef91c0a2" />


#### example  

<img width="347" alt="image" src="https://github.com/user-attachments/assets/550d8b89-2bef-432c-a0d3-6a4d111e97ba" />


#### Implementation 

<img width="624" alt="image" src="https://github.com/user-attachments/assets/94d3552e-f8ce-4b50-9593-66f5ee4d17a1" />

## Importing

<img width="361" alt="image" src="https://github.com/user-attachments/assets/46bc8fc7-42c9-4be5-a911-eba6a8445bce" />

## Package vs. Java Package

<img width="615" alt="image" src="https://github.com/user-attachments/assets/d0596e9a-23e5-420c-82f1-417fd9089990" />

## How Proto Works

<img width="1402" alt="image" src="https://github.com/user-attachments/assets/ca9aa7db-00fb-4bec-9d46-d6ff6af6df5f" />

## Message Format Changes

If you are applying a message format change, use the `reserved` keyword.

<img width="258" alt="image" src="https://github.com/user-attachments/assets/e9aee8f0-73a4-42ae-b672-f081d8aa6194" />

## Best Pratice

<img width="517" alt="image" src="https://github.com/user-attachments/assets/dc239fcb-b459-4295-a1c3-5095ebb3bed1" />


## gRPC Introduction

<img width="739" alt="image" src="https://github.com/user-attachments/assets/f6e3b10f-3738-41ac-abc3-d675e2776374" />

## Communication parrterns 

<img width="718" alt="image" src="https://github.com/user-attachments/assets/889d69d7-9eab-403c-a43f-27da4d0089dd" />

