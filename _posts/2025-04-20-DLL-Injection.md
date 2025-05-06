---
title: DLL Injection
author: 
date: 2025-04-20 00:34:00 +0800
categories: [Reversing]
---

## DLL 이란?
---
DLL은 ‘Dynamic Link Library(동적 링크 라이브러리)’의 약자로 여러 프로그램에서 동시에 사용할 수 있는 코드와 데이터를 포함하는 동적 라이브러리이다.

아래 사진의 kernel32.dll은 Visual Studio에서 만든 프로그램을 실행할 때 로드되는 dll이다. kernel32.dll에는 프로그램을 개발할 때 필요한 함수가 내장되어 있다. 그래서 kernel32.dll에서 필요한 함수를 가져와서 사용한다.

![dll_img](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FO0kfo%2FbtsMGdc12Gp%2FL7xnhCC1roNRnjJbzbf0d1%2Fimg.png)


<br>
<br>


## DLL Injection 이란?
---
![dll_process](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2Fcf61OH%2FbtsMEOS1AMV%2FHYK8zNu3OX7S8EUez0SBRk%2Fimg.png)

DLL Injection은 Target Process의 주소 공간에 myhack.dll을 강제로 로드하고 실행시키는 기법이다.

* InjectDll.exe → 타겟 프로세스를 조작하는 프로그램
* Target Process → myhack.dll이 실행될 대상 프로세스
* Kernel32.dll → LoadLibrary 함수의 주소를 포함한 DLL

그림을 요약하면 위와 같다. 나머지는 DLL Injection 실행 과정에서 설명할 예정이다.


<br>
<br>


## DLL Injection 실행 과정
---

1. #### OpenProcess
```c++
hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
```
DLL Injection을 수행하기 위해서는 먼저 Target Process에 접근할 수 있는 핸들을 확보해야 한다.
이를 이용해 프로세스의 제어권을 얻을 수 있다.<br>
OpenProcess는 Target Process의 PID로 핸들(HANDLE)을 얻는 함수이다.
여기서 핸들은 프로세스의 주소가 아니라 제어권과 유사한 개념이다.
핸들을 사용해 메모리를 할당하거나 스레드를 생성해야 하므로, 모든 액세스 권한을 허용하도록 설정한다.<br><br>


2. #### VirtualAllocEx
```c++
VMAddress = VirtualAllocEx(hProcess, NULL, dllSize, MEM_COMMIT, PAGE_READWRITE);
```
1번에서 구한 핸들을 이용해서 Target Process에 메모리를 할당할 수 있는 권한을 얻었다.
이제 VirtualAllocEx로 DLL을 실행시킬 공간을 확보한다.<br>
이 함수는 dllSize만큼 메모리를 할당하며, 여기서는 DLL 경로의 길이만큼 메모리를 할당했다.
할당된 메모리에는 읽기 및 쓰기 권한을 부여해야 한다. 이 권한이 있어야 DLL 경로를 메모리에 쓸 수 있다.
VirtualAllocEx은 할당된 메모리의 시작 주소를 반환한다.<br><br>


3. #### WriteProcessMemory
```c++
WriteProcessMemory(hProcess, VMAddress, (LPVOID)dllPath, dllSize, NULL);
```
2번에서 dll을 실행할 공간을 확보했으니 WriteProcessMemory로 어떤 DLL을 실행시킬지 지정해야 한다.<br>
WriteProcessMemory는 메모리에 DLL의 경로를 쓰는 역할을 한다.
2번에서 할당된 메모리의 시작 주소부터 dllSize만큼 DLL이 위치한 경로를 쓴다.<br><br>


4. #### GetModuleHandle
```c++
hKernel = GetModuleHandle(L"Kernel32.dll");
```
이제 loadLibrary 함수의 주소를 찾아야 한다.
이를 위해서 GetModuleHandle로 Kernel32.dll의 모듈 핸들(HMODULE)을 얻어야 한다.<br>
모듈 핸들은 DLL 또는 실행 파일(EXE)이 메모리에 로드된 주소를 의미한다.
Kernel32.dll에는 LoadLibrary의 시작 주소가 포함되어 있기 때문에 Kernel32.dll의 주소가 필요하다.
* hProcess를 사용하지 않은 이유<br>
        * 지금까지의 과정에서는 Target Process의 핸들(hProcess)을 사용해 작업을 수행했다.
하지만 5번 과정에서 hProcess를 사용하지 않았다. 그 이유는 DLL의 주소는 모든 프로세스에서 동일하기 때문이다.<br>
즉, Kernel32.dll은 모든 프로세스에서 같은 주소에 로드된다.
Target Process에서 구할 필요 없이 자신의 프로세스에서 구해도 문제없다.
<br><br>

5. #### GetProcAddress
```c++
LoadLibraryAddr = GetProcAddress(hKernel, "LoadLibraryW");
```
4번에서 Kernel32.dll의 모듈 핸들을 얻었으니 GetProcAddress로 loadLibrary 함수의 주소를 찾을 수 있다.<br>
GetProcAddress는 DLL에서 원하는 함수 또는 변수의 주소를 찾을 수 있는 함수이다.
위의 코드는 Kerenl32.dll에서 LoadLibrary의 주소를 찾는 코드이다.
LoadLibrary의 주소를 찾는 이유는 이 함수를 사용해 원하는 DLL을 프로세스에 로드할 수 있기 때문이다.<br>
즉, LoadLibrary를 실행하면 myhack.dll을 Target Process에 로드할 수 있다.<br><br>


6. #### CreateRemoteThread
```c++
hThread = CreateRemoteThread(hProcess, NULL, dllSize, (LPTHREAD_START_ROUTINE)LoadLibraryAddr, 
			VMAddress, 0, NULL);
```
마지막으로 스레드를 만들고 실행시키면 Dll Injection은 마무리된다.
스레드는 프로세스 내에서 실제로 작업을 수행하는 주체를 말한다.<br>
CreateRemoteThread를 사용해 Target Process 내부에 새로운 스레드를 생성하고, 이 스레드가 myhack.dll을 실행하도록 설정해야 한다.
CreateRemoteThread의 주요 인자는 다음과 같다

    * hProcess: Target Process 내부에 스레드를 생성한다.
    * LoadLibraryAddr: LoadLibrary 함수의 주소를 스레드의 시작 주소로 설정하여, 스레드가 실행되면 myhack.dll을 로드하게 만든다.
    * VMAddress: myhack.dll의 경로가 저장된 메모리 주소를 전달하여, 스레드가 어떤 DLL을 실행할지 알 수 있도록 한다.
    * 0: 스레드를 즉시 실행하도록 설정한다.<br>

    이제 CreateRemoteThread를 실행하면 Target Process 내부에서 myhack.dll이 로드된다. 이로써 DLL Injection 과정이 마무리된다.



<br>
<br>



## PID (ProcessID)
---
PID (ProcessID)는 운영체제에서 프로세스를 식별하기 위해 부여하는 번호이다.<br>
OpenProcess의 인자로 PID가 필요하다. 기존의 DLL Injection에서는 tasklist 명령어를 사용하는 등의 방법으로 직접 PID를 알아냈다. DLL Injection을 할 때마다 PID를 구하는 것은 번거롭다. 이를 위해 GetProcessID 함수를 만들어서 이 과정을 생략하겠다.
![tasklist](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FbsUSNI%2FbtsNKs2QQBQ%2FQkBJZ0Yz50JDK90XVnbWik%2Fimg.png)


<br>
<br>


## GetProcessID 실행 과정
---
1. #### CreateToolhelp32Snapshot
```c++
hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
```
프로세스의 PID를 알아내기 위해서는, 먼저 해당 PID가 포함된 프로세스 정보를 얻어야 한다. 
이를 위해 시스템에서 현재 실행 중인 모든 프로세스 목록을 가져온 뒤, 그중에서 Target Process를 찾아야 한다.<br>
CreateToolhelp32Snapshot는 이러한 모든 프로세스 정보를 스냅샷 형태로 가져오는 데 사용된다.<br><br>


2. #### sizeof(PROCESSENTRY32)
```c++
pe32.dwSize = sizeof(PROCESSENTRY32);
```
1번에서 프로세스의 정보를 가져왔다면 가져온 정보를 저장해야 한다.
하지만 프로세스 정보를 저장하기 전에 PROCESSENTRY32 구조체의 멤버인 dwSize를 초기화해야 한다.
pe32는 PROCESSENTRY32 구조체이며, 프로세스 항목들을 구조체 멤버로 가지고 있다.<br><br>


3. #### Process32First
```c++
Process32First(hSnapshot, &pe32);
```
2번에서 PROCESSENTRY32 구조체 멤버인 dwSize를 초기화했으니 프로세스의 정보를 저장할 수 있다.
Process32First는 스냅샷의 첫번째 엔트리에 있는 정보를 가져오고 이를 PROCESSENTRY32 구조체에 저장한다.<br><br>


4. #### wcscmp
```c++
if (wcscmp(pe32.szExeFile, processName) == 0) { 
    CloseHandle(hSnapshot);
	return pe32.th32ProcessID; 
}
```
3번에서 프로세스의 정보를 저장했다면 구조체에 저장된 프로세스가 찾고 있는 프로세스와 동일한지 확인해야 한다.<br>
wcscmp는 WCHAR의 문자열을 비교하는 함수이다.
pe32에 저장된 프로세스와 찾고 있는 프로세스의 이름이 동일하다면 pe32에 저장된 PID를 반환한다.<br><br>


5. #### Process32Next
```c++
do {
    ...
} while (Process32Next(hSnapshot, &pe32));
```
4번에서 if문을 통과하지 못 했다면 다음 프로세스의 정보를 가져와야 한다. 
Process32Next는 스냅샷의 다음 엔트리에 있는 정보를 가져오고 이를 PROCESSENTRY32 구조체에 저장한다.
4번의 if문을 통과할 때까지 Process32Next를 호출해서 다음 정보를 저장하는 행위를 반복한다.



<br>
<br>

## DLL Injection 전체 코드
---
```c++
#include <Windows.h>
#include <stdio.h>
#include <tlhelp32.h>
#include "TCHAR.h"



DWORD GetProcessID(const WCHAR* processName) {
	HANDLE hSnapshot;
	PROCESSENTRY32 pe32;

	hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0); 

	pe32.dwSize = sizeof(PROCESSENTRY32); 

	Process32First(hSnapshot, &pe32);

	do {
		if (wcscmp(pe32.szExeFile, processName) == 0) { 
			CloseHandle(hSnapshot);
			return pe32.th32ProcessID; 
		}
	} while (Process32Next(hSnapshot, &pe32)); 

	CloseHandle(hSnapshot);

	return 0;
}

BOOL InjectDLL(DWORD PID, LPCTSTR dllPath) {

	HANDLE hProcess = NULL, hThread = NULL;
	HMODULE hKernel = NULL;
	LPVOID VMAddress = NULL;
	SIZE_T dllSize = (wcslen(dllPath) + 1) * sizeof(TCHAR); 
	FARPROC LoadLibraryAddr;

	hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);

	VMAddress = VirtualAllocEx(hProcess, NULL, dllSize, MEM_COMMIT, PAGE_READWRITE);

	WriteProcessMemory(hProcess, VMAddress, (LPVOID)dllPath, dllSize, NULL);

	hKernel = GetModuleHandle(L"Kernel32.dll");

	LoadLibraryAddr = GetProcAddress(hKernel, "LoadLibraryW");

	hThread = CreateRemoteThread(hProcess, NULL, dllSize, (LPTHREAD_START_ROUTINE)LoadLibraryAddr, VMAddress, 1, NULL);

	WaitForSingleObject(hThread, INFINITE);

	CloseHandle(hThread);
	CloseHandle(hProcess);

	return TRUE;
}

int _tmain(int argc, TCHAR* argv[])
{
	DWORD PID = GetProcessID(argv[1]);
	printf("PID : %u\n", PID);

	InjectDLL(PID, argv[2]);
	
	return 0;
}
```