# Reading and Writing on STM32 MCU Flash Memory
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Reading and Writing on STM32 MCU Flash Memory](#reading-and-writing-on-stm32-mcu-flash-memory)
  - [Memory Divided by Pages](#memory-divided-by-pages)
    - [Flash Module Organization (medium-density devices)](#flash-module-organization-medium-density-devices)
    - [Flash_Write_Data](#flash_write_data)
        - [1. Computing the Number of Words](#1-computing-the-number-of-words)
        - [2. Unlocking Flash](#2-unlocking-flash)
        - [3. Erasing Flash](#3-erasing-flash)
        - [4. Programming(Writing) Flash](#4-programmingwriting-flash)
        - [5. Lock Flash](#5-lock-flash)
    - [Flash_Read_Data](#flash_read_data)
  - [Memory Divided by Sectors](#memory-divided-by-sectors)
    - [Flash_Write_Data](#flash_write_data-1)
      - [Erasing the Flash](#erasing-the-flash)
      - [Programming(Writing) Flash](#programmingwriting-flash)
  - [Reading Data from the Memory](#reading-data-from-the-memory)
  - [Converting Stored Data to String](#converting-stored-data-to-string)
  - [External References](#external-references)

<!-- /code_chunk_output -->

Every microcontroller had an allocated flash-memory in which the firmware are stored. The advantage of the flash-memory is that it retains its data even the power is disconnected. It is ideal for constants to be stored which vital for the firmware's operation.

## Memory Divided by Pages

On this example we will use the development board called "blue pill" and will use STM32CubeIDE as text editor. The blue pill use STM32F103C8 MCU. 

### Flash Module Organization (medium-density devices)
STM32F103C8 belongs to the medium density devices of ST. The table below shows that STM32F103C8 had 128 pages of 1 Kbyte memory block starting from 0x0800 0000 to 0x0801 FFFF.


|      Block       |    Name     |       Base adresses       | Size (bytes) |
|:----------------:|:-----------:|:-------------------------:|:------------:|
| Main <br> memory |   Page 0    | 0x0800 0000 - 0x0800 03FF |   1 Kbyte    |
|        ^         |   Page 1    | 0x0800 0400 - 0x0800 07FF |   1 Kbyte    |
|        ^         |   Page 2    | 0x0800 0800 - 0x0800 0BFF |   1 Kbyte    |
|        ^         |   Page 3    | 0x0800 0C00 - 0x0800 0FFF |   1 Kbyte    |
|        ^         |   Page 4    | 0x0800 1000 - 0x0800 13FF |   1 Kbyte    |
|        ^         | -<br>-<br>- |        -<br>-<br>-        | -<br>-<br>-  |
|        ^         |  Page 127   | 0x0801 FC00 - 0x0801 FFFF |   1 Kbyte    |


If we want to save variable/constants into the flash-memory, other than the main firmware, it is best to start at the very last page 0x0801 FC00 going up in order to avoid overwritting the firmware itself.

### Flash_Write_Data
This function writes 32-bit of data into a specified flash memory address. It has two arguments which are as shown on the code snippet below:
```c
uint32_t Flash_Write_Data (uint32_t StartPageAddress, uint32_t * data){
	...
	return 0;
}
```
- **StartPageAddress** - the starting address of the page in flash-memory which where you want to write.
- **data** - the address of the 32-bit data to be written into the flash memory.

The flow chart below shows the process the function performs when it is called.

```mermaid
graph LR;
A([Start]) --> B[/Compute Number of Words/];
B[/Compute Number of Words/] --> C[/Unlock Flash/];
C[/Unlock Flash/] --> D[/Erase Flash/];
D[/Erase Flash/] --> E[/Program Flash/];
E[/Program Flash/] --> F[/Lock Flash/];
F[/Lock Flash/] --> G([End]);
```


##### 1. Computing the Number of Words 
The function first computes how many words the data have. On this case a word has four bytes into it. 

The snippet below computes the number word by adding the qoutient of the lenght data(using _strlen(data)_ function) divided by four , and added by one, if the length of the data has a remainder (divided by four) and zero if none.

```c
...
int numberofwords = (strlen(data)/4) + ((strlen(data) % 4) != 0);
```
For example a "_Hello World_" string that has data length of 11, would have a three words as output of this process. This is because 11/4 = 2, with the _"rld"_  as a remainder, which would lead to  2 + 1 = 3. 

The table below maps how many word the example string have.

| Word    | > | > | > | 1 | > | > | > | 2 | > | > | >  | 3  |
|---------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:--:|:--:|
| data[ ] | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
| char    | H | e | l | l | o | _ | W | o | r | l | d  | _  |

##### 2. Unlocking Flash 
Next we must unlock the flash memory for modification by using the code below.
```c
HAL_FLASH_Unlock();
```
##### 3. Erasing Flash
After the flash-memory has been unlock, the function  erases the area unto which the data will be written, but we have to define FLASH_EraseInitTypeDef type first.

```c
static FLASH_EraseInitTypeDef EraseInitStruct;
uint32_t PAGEError;
...
//Declare structure variables
uint32_t StartPage 			= GetPage(StartPageAddress);
uint32_t EndPageAdress 		= StartPageAddress + numberofwords*4;
uint32_t EndPage 			= GetPage(EndPageAdress);
uint32_t NumberOfPages		= ((EndPage - StartPage)/FLASH_PAGE_SIZE) +1;
//Fill EraseInit structure
EraseInitStruct.TypeErase   = FLASH_TYPEERASE_PAGES;
EraseInitStruct.PageAddress = StartPage;
EraseInitStruct.NbPages     = NumberOfPages;
//capture error
if (HAL_FLASHEx_Erase(&EraseInitStruct, &PAGEError) != HAL_OK){
	return HAL_FLASH_GetError ();
}
...
```

The *GetPage()* function gets the page number of StartPageAddress for it is not always we want to store our variable at the start address of the page. Remember that every page has 1KByte (1024 bytes) of data, which has a lot if we want only to store numeric values (even some character arrays.)

The code below returns the starting page address of any address that you will pass on its arguments.
```c
static uint32_t GetPage(uint32_t Address){
	for (int indx = 0; indx < 128; indx++){
		if((Address < (0x08000000 + (1024 *(indx+1)))) && (Address >= (0x08000000 + 1024*indx))){
			return (0x08000000 + 1024*indx);
		}
	}
	return -1;
}
```
**EraseInit Structure Variable Declaration**
- **StartPage** - is the page name or number in the memory block in which we want to store the _data_. This can be computed with the _GetPage()_ function.
The _GetPage()_ function is another function within the library. In our example the start page address of _0x0801 FC00_ would have a page number of **127**.
- **EndPageAddress** - is the after page address on the memory on which the data will be written grouped in fours. The _EndPageAddress_ is computed by adding the _StartPageAddress_ with the _numberofwords_  multiplied by four(4). In our example, the last page page in which the "Hello World" would be written is 0x0801 FC0B(plus with the nullafter the character 'd'). The end address would be 0x0801 FC00 + (3*4) = **0x0801 FC0C**. <br>
<img src="/endpageaddress.png" width="95%">
<br>
- **EndPage** - is the page name or number in the memory block in which the last bit of data will be stored. This can be computed also with the GetPage() function.

**EraseInit Structure**
the following structures are required on erasing the flash-memory.
- **TypeErase** - Mass erase or page erase. On our example we are using page erase, so the value would be _FLASH_TYPEERASE_PAGES_.
- **PageAddress** -  The initial flash page address to be erased. This has been computed and saved in our _StartPage_ variable.
- **NbPages** - Number of pages to be erased. On our example, this was computed by adding the quotient of (EndPage - StartPage) and FLASH_PAGE_SIZE(1024) by one (1).
	

##### 4. Programming(Writing) Flash
After the page area has been erased, we can now write the data into the flash memory. The snippet below writes the data into the flash-memory, word by word. this is determined by the _HAL_FLASH_Program's_ argument **FLASH_TYPEPROGRAM_WORD**.
If the writing process is OK, we increment the _ctr_, and  add 4 to the _StartPageAddress_, else an error is captured.


```C++
int ctr = 0;
while (ctr < numberofwords){
	if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, StartPageAddress, data[ctr]) == HAL_OK){
		StartPageAddress += 4;
		ctr++;
	}
	else{
		return HAL_FLASH_GetError ();
	}
}
```
##### 5. Lock Flash
After the writing process, we must lock the flash by running the code below to protect the flash from unwanted operation.

```c
...
HAL_FLASH_Lock();
```
### Flash_Read_Data
This function reads data from the flash memory and has the following arguments.

```C++
void Flash_Read_Data (uint32_t StartPageAddress, __IO uint32_t * data){
	...
}
```
- **StartPageAddress** - the starting address of the page in flash-memory which where we want to read data.
- **data** - the address of the 32-bit variable where we want to store the data.

```C++
while (1){
	*var = *(__IO uint32_t *)StartPageAddress;
	if (*var == 0xffffffff){
		*var = '\0';
		break;
	}
	StartPageAddress += 4;
	var++;
}
```
The program will store the value into the _*var_  word by word, determined by incrementing _StartPageAddress_ by 4, until value is equal to 0xFFFFFFFF, in which it replaces it with \0. 

Afterwhich it exits the function. The value of _StartPageAddress_ now is tored on the address value of _*var_.
## Memory Divided by Sectors
Some ST's MCU are mapped with sectors instead of pages. These sectors differs in sizes. On this example we are using Nucleo F401RE with STM32F401RE as its core MCU. The table below shows the flash-memory mapping of the said MCU.


| Block       |     Name      | Block base addresses      | Size       |
|-------------|:-------------:|---------------------------|----------- |
| Main memory |   Sector 0    | 0x0800 0000 - 0x0800 3FFF | 16 Kbytes  |
| ^           |   Sector 1    | 0x0800 4000 - 0x0800 7FFF | 16 Kbytes  |
| ^           |   Sector 2    | 0x0800 8000 - 0x0800 BFFF | 16 Kbytes  |
| ^           |   Sector 3    | 0x0800 C000 - 0x0800 FFFF | 16 Kbytes  |
| ^           |   Sector 4    | 0x0801 0000 - 0x0801 FFFF | 64 Kbytes  |
| ^           |   Sector 5    | 0x0802 0000 - 0x0803 FFFF | 128 Kbytes |
| ^           |   Sector 6    | 0x0804 0000 - 0x0805 FFFF | 128 Kbytes |
| ^           |   Sector 7    | 0x0806 0000 - 0x0807 FFFF | 128 Kbytes |
| >           | System memory | 0x1FFF 0000 - 0x1FFF 77FF | 30 Kbytes  |
| >           |   OTP Area    | 0x1FFF 7800 - 0x1FFF 7A0F | 528 bytes  |
| >           | Option bytes  | 0x1FFF C000 - 0x1FFF C00F | 16 bytes   |

### Flash_Write_Data
This function writes a data into the specified sector address on flash memory. The structure is the same as former but takes sector address instead of page address.

```c
uint32_t Flash_Write_Data (uint32_t StartSectorAddress, uint32_t * data){
	...
	return 0;
}
```
The function has two arguments which are both are addresses in the memory.
- **StartSectorAddress** - the starting address of the sector into which the data will be written.
- **data** - the address of the data to be stored in the flash-memory.
The flow of the function is the same as the former function but only differs on the *erasing of flash* and *programming of flash*, which will only be discussed on this section. Refer to the previous section for the full function flow.

The flow of the function is the same as the flow of the function on writing on a memory divided by pages stated above. I will just point out the difference in this section.

#### Erasing the Flash

```c
...
static FLASH_EraseInitTypeDef EraseInitStruct;
uint32_t SECTORError;
int ctr = 0;

uint32_t StartSector		  = GetSector(StartSectorAddress);
uint32_t EndSectorAddress	  = StartSectorAddress + numberofwords*4;
uint32_t EndSector 			  = GetSector(EndSectorAddress);
uint32_t NumberOfSectors 	  = (EndSector - StartSector) + 1;
//Fill EraseInit structure
EraseInitStruct.TypeErase     = FLASH_TYPEERASE_SECTORS;
EraseInitStruct.VoltageRange  = FLASH_VOLTAGE_RANGE_3;
EraseInitStruct.Sector        = StartSector;
EraseInitStruct.NbSectors     = NumberOfSector

if (HAL_FLASHEx_Erase(&EraseInitStruct, &SECTORError) != HAL_OK){
	return HAL_FLASH_GetError ();
}
```
The difference between the functions are, first is on the variable declaration, the "Page" was replaced with "Sectors" like *StartPageAddress* was replaced with *StartSectorAddress*. Then the EraseInit requires an additional VoltageRange structure for devices STM32F4xx series. This voltage range is the device voltage range which defines the erase parallelism with the following must have values below. 

Since the nucleo's mcu operates on 3.3 volt range, the voltage range that must be selected *FLASH_VOLTAGE_RANGE_3* which has an operating range of 2.7V to 3.6V.

```c
#define FLASH_VOLTAGE_RANGE_1        0x00000000U  /*!< Device operating range: 1.8V to 2.1V                */
#define FLASH_VOLTAGE_RANGE_2        0x00000001U  /*!< Device operating range: 2.1V to 2.7V                */
#define FLASH_VOLTAGE_RANGE_3        0x00000002U  /*!< Device operating range: 2.7V to 3.6V                */
#define FLASH_VOLTAGE_RANGE_4        0x00000003U  /*!< Device operating range: 2.7V to 3.6V + External Vpp */
```

#### Programming(Writing) Flash
This process is the same as with the former function except for the second argument on the *HAL_FLASH_Program()* function which takes in the *StartSectorAddress* instead of the *StartPageAddress*.

```c
...
while (ctr < numberofwords){
	if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, StartSectorAddress, data[ctr]) == HAL_OK){
		StartSectorAddress += 4;  
		ctr++;
	}
	else{
		return HAL_FLASH_GetError ();
	}
}
```
## Reading Data from the Memory
The code below retrieves the data stored at the flash-memory which can be applied to both devices whose memory are divided by either pages or sectors. The function has two arguments, *StartPageAddress* which is the address of the data we will retrieve. Then the *data*, which is the address of the variable which we want the data to store with.
```c
void Flash_Read_Data (uint32_t StartPageAddress, __IO uint32_t * data){
	while (1){
		*data = *(__IO uint32_t *)StartPageAddress;
		if (*data == 0xffffffff){
			*data = '\0';
			break;
		}
		StartPageAddress += 4;
		data++;
	}
}
```
Inside the function is a loop which passess the data from one address to another word by word. This are determined by the increment in *StartPageAddress* by four(4).

## Converting Stored Data to String
To read the stored string data on the flash memory address, we need to convert the data into strings using the function below.
 


```c
void Convert_To_Str (uint32_t *data, char *str){
	int numberofbytes = ((strlen(data)/4) + ((strlen(data) % 4) != 0)) *4;

	for (int i=0; i<numberofbytes; i++){
		str[i] = data[i/4]>>(8*(i%4));
	}
}
```
The function first computes the number of bytes the data has but on the multiple of fours. In our example "Hello World" which has 11 characters, would have a computed value of 12.

```c
 numberofbytes = ((strlen(data)/4) + ((strlen(data) % 4) != 0)) *4
 numberofbytes = ((11/4) + (11%1) !=0 ) * 4 
 numberofbytes = ((2) + 1) * 4
 numberofbytes = 12
```
Since the process of reading is by word(4 bytes), *data[]* are being divided into four groups, means 
```c
data[0] = "Hell"
data[1] = "o Wo"
data[2] = "rld "
```
The rest of the operation's value are shown on the table below.
| i  | str[i] | [i/4] | data[i/4] | >>(8*(i%4)) | data[i/4]>>(8*(i%4)) |
|----|:------:|:-----:|-----------|-------------|----------------------|
| 0  | 'H'    | 0     | "lleH"    | >>0         | 'H'                  |
| 1  | 'e'    | 0     | "lleH"    | >>8         | 'e'                  |
| 2  | 'l'    | 0     | "lleH"    | >>16        | 'l'                  |
| 3  | 'l'    | 0     | "lleH"    | >>24        | 'l'                  |
| 4  | 'o'    | 1     | "oW o"    | >>0         | 'o'                  |
| 5  | ' '    | 1     | "oW o"    | >>8         | ' '                  |
| >  | >      | >     | >         | >           | -                    |
| >  | >      | >     | >         | >           | -                    |
| >  | >      | >     | >         | >           | -                    |
| 11 | 'd'    | 2     | " dlr"    | >>16        | 'd'                  |
| 12 | null   | 2     | "dlr"     | >>24        | ' '                  |

finally

## External References
- [Blue Pill](https://stm32duinoforum.com/forum/wiki_subdomain/index_title_Blue_Pill.html)
- [STM32F10xxx Flash memory microcontrollers](https://www.st.com/resource/en/programming_manual/cd00283419-stm32f10xxx-flash-memory-microcontrollers-stmicroelectronics.pdf)
- [FLASH Programming in STM32](https://controllerstech.com/flash-programming-in-stm32/)
