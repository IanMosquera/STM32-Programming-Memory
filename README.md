### Reading and Writing on Memory
<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Reading and Writing on Memory](#reading-and-writing-on-memory)
  - [Flash Module Organization (medium-density devices)](#flash-module-organization-medium-density-devices)
  - [Flash_Write_Data](#flash_write_data)
    - [Function Flow](#function-flow)
      - [1. Compute the Number of Words](#1-compute-the-number-of-words)
      - [2. Unlock Flash](#2-unlock-flash)
      - [3. Erase Flash](#3-erase-flash)
      - [4. Program Flash](#4-program-flash)
      - [5. Lock Flash](#5-lock-flash)
  - [Flash_Read_Data](#flash_read_data)
- [External References](#external-references)

<!-- /code_chunk_output -->

Every microcontroller had an allocated flash-memory in which the firmware are stored. The advantage of the flash-memory is that it retains its data even the power is disconnected. It is ideal for constants to be stored which vital for the firmware's operation.

On this document we will use the development board called "blue pill" and will use STM32CubeIDE as text editor. The blue pill use STM32F103C8 MCU.

#### Flash Module Organization (medium-density devices)
STM32F103C8 belongs to the medium density devices of ST. The table below shows that STM32F103C8 had 128 pages of 1 Kbyte memory block starting from 0x0800 0000 to 0x0801 FFFF.


| Block | Name | Base adresses | Size (bytes)|
|:--:|:--:|:--:|:--:|:--:|
| Main <br> memory |Page 0|0x0800 0000 - 0x0800 03FF|1 Kbyte|
|^|Page 1|0x0800 0400 - 0x0800 07FF|1 Kbyte|
|^|Page 2|0x0800 0800 - 0x0800 0BFF|1 Kbyte|
|^|Page 3|0x0800 0C00 - 0x0800 0FFF|1 Kbyte|
|^|Page 4|0x0800 1000 - 0x0800 13FF|1 Kbyte|
|^|-<br>-<br>-|-<br>-<br>-|-<br>-<br>-|
|^|Page 127|0x0801 FC00 - 0x0801 FFFF|1 Kbyte|


If we want to save variable/constants into the flash-memory, other than the main firmware, it is best to start at the very last page 0x0801 FC00 going up in order to avoid overwritting the firmware itself.

#### Flash_Write_Data
This function writes 32-bit of data into a specified flash memory address. It has two arguments which are as shown on the code snippet below:
```c++
uint32_t Flash_Write_Data (uint32_t StartPageAddress, uint32_t * data){
	...
	return 0;
}
```
- **StartPageAddress** - the starting address of the page in flash-memory which where you want to write.
- **data** - the address of the 32-bit data to be written into the flash memory.

##### Function Flow
The flow chart below shows the process the function performs when it is called.

```mermaid
graph LR
A([Start]) --> B[/Get Number of Words/]
B[/Get Number of Words/] --> C[/Unlock Flash/]
C[/Unlock Flash/] --> D[/Erase Flash/]
D[/Erase Flash/] --> E[/Program Flash/]
E[/Program Flash/] --> F[/Lock Flash/]
F[/Lock Flash/] --> G([End])
```


###### 1. Compute the Number of Words 
The function first computes how many words the data have. The snippet below computes the number word by adding the qoutient of the lenght data(using _strlen(data)_ function) divided by four , and added by one, if the length of the data has a remainder (divided by four) and zero if none.

```C { output="html"}
...
int numberofwords = (strlen(data)/4) + ((strlen(data) % 4) != 0);
```
For example a "_Hello World_" string that has character length of 11, would have a three words. This is because 11/4 = 2, with the _"rld"_  as a remainder, which would lead to  2 + 1 = 3. The table below maps how many word the example string have.

|Word|>|>|>|1|>|>|>|2|>|>|>|3|
|--|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
|data[ ]|0|1|2|3|4|5|6|7|8|9|10|11|
|char|H|e|l|l|o| _ |W|o|r|l|d|_|

###### 2. Unlock Flash 
Next we must unlocks the flash memory for modification by using the code below.
```C++
HAL_FLASH_Unlock();
```
###### 3. Erase Flash
After the flash-memory has been unlock, the function  erases the area unto which the data will be written. First we have to declare the FLASH_EraseInitTypeDef.

```C++
static FLASH_EraseInitTypeDef EraseInitStruct;
uint32_t PAGEError;
...
//Declare structure Variables
uint32_t StartPage = GetPage(StartPageAddress);
uint32_t EndPageAdress = StartPageAddress + numberofwords*4;
uint32_t EndPage = GetPage(EndPageAdress);
uint32_t NumberOfPage = ((EndPage - StartPage)/FLASH_PAGE_SIZE) +1;

//Fill EraseInit structure
EraseInitStruct.TypeErase   = FLASH_TYPEERASE_PAGES;
EraseInitStruct.PageAddress = StartPage;
EraseInitStruct.NbPages     = NumberOfPage;

//capture error
if (HAL_FLASHEx_Erase(&EraseInitStruct, &PAGEError) != HAL_OK){
	return HAL_FLASH_GetError ();
}
...
```
**Variable Declaration**
- **StartPage** - is the page name or number in the memory block in which we want to store the _data_. This can be computed with the _GetPage()_ function.
The _GetPage()_ function is another function within the library. In our example the start page address of _0x0801 FC00_ would have a page number of **127**.
- **EndPageAddress** - is the after page address on the memory on which the data will be written grouped in fours. The _EndPageAddress_ is computed by adding the _StartPageAddress_ with the _numberofwords_  multiplied by four(4). In our example, the last page page in which the "Hello World" would be written is 0x0801 FC0B(plus with the nullafter the character 'd'). The end address would be 0x0801 FC00 + (3*4) = **0x0801 FC0C**.
<img src="/endpageaddress.png" width="95%">
- **EndPage** - is the page name or number in the memory block in which the last bit of data will be stored. This can be computed also with the GetPage() function.

**Erasing the Flash-Memory**
- **TypeErase** - Mass erase or page erase. On our example we are using page erase, so the value would be _FLASH_TYPEERASE_PAGES_.
- **PageAddress** -  the initial FLASH page address to be erased. This has been computed and saved in our _StartPage_ variable.
- **NbPages** - number of pages to be erased. On our example, this was computed by adding the quotient of (EndPage - StartPage) and FLASH_PAGE_SIZE(1024) by one (1).
	

###### 4. Program Flash
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
###### 5. Lock Flash
After the writing process, we must lock the flash by running the code below to protect the flash from unwanted operation.

```C++
...
HAL_FLASH_Lock();
```
#### Flash_Read_Data
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

### External References
- [Blue Pill](https://stm32duinoforum.com/forum/wiki_subdomain/index_title_Blue_Pill.html)
- [STM32F10xxx Flash memory microcontrollers](https://www.st.com/resource/en/programming_manual/cd00283419-stm32f10xxx-flash-memory-microcontrollers-stmicroelectronics.pdf)
- [FLASH Programming in STM32](https://controllerstech.com/flash-programming-in-stm32/)
