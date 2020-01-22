# DLL

### 实验过程

**待分析源代码**

```c
#include <windows.h>
#include <tchar.h>
#include <strsafe.h>

#define MAX_THREADS 10	//创建的线程数
#define BUF_SIZE 255
// #define MULTI_THREAD

DWORD WINAPI MyThreadFunction(LPVOID lpParam);

// Sample custom data structure for threads to use.
// This is passed by void pointer so it can be any data type
// that can be passed using a single void pointer (LPVOID).
typedef struct MyData {
	int val1;
	int val2;
} MYDATA, * PMYDATA;

// 支持unicode, main函数的别名
int _tmain()
{
	PMYDATA pDataArray[MAX_THREADS];
#ifdef MULTI_THREAD
	DWORD   dwThreadIdArray[MAX_THREADS];
	HANDLE  hThreadArray[MAX_THREADS];
#endif

	// Create MAX_THREADS worker threads.

	DWORD start = GetTickCount();
	//The return value is the number of milliseconds that have elapsed since the system was started.

	for (int i = 0; i < MAX_THREADS; i++)
	{
		// Allocate memory for thread data.

		pDataArray[i] = (PMYDATA)malloc(sizeof(MYDATA));

		if (pDataArray[i] == NULL)
		{
			// If the array allocation fails, the system is out of memory
			// so there is no point in trying to print an error message.
			// Just terminate execution.
			ExitProcess(2);
		}

		// Generate unique data for each thread to work with.

		pDataArray[i]->val1 = i;
		pDataArray[i]->val2 = i + 100;

		// Create the thread to begin execution on its own.

#ifdef MULTI_THREAD

		hThreadArray[i] = CreateThread(
			NULL,                   // default security attributes
			0,                      // use default stack size
			MyThreadFunction,       // thread function name
			pDataArray[i],          // argument to thread function
			0,                      // use default creation flags
			&dwThreadIdArray[i]);   // returns the thread identifier

		// Check the return value for success.
		// If CreateThread fails, terminate execution.
		// This will automatically clean up threads and memory.

		if (hThreadArray[i] == NULL)
		{
			printf("CreateThread Error(%d)", GetLastError());
			ExitProcess(3);
		}
#else
		MyThreadFunction(pDataArray[i]);
#endif
	} // End of main thread creation loop.

	// Wait until all threads have terminated.
	// TRUE表示所有都要执行完毕才返回, FALSE只要有任意一个执行完毕就返回
	// INFINTE 无限等待
#ifdef MULTI_THREAD
	WaitForMultipleObjects(MAX_THREADS, hThreadArray, TRUE, INFINITE);
#endif

	DWORD end = GetTickCount();
	printf("tick count: %d\n", end - start);

	// Close all thread handles and free memory allocations.
#ifdef MULTI_THREAD

	for (int i = 0; i < MAX_THREADS; i++)
	{
		CloseHandle(hThreadArray[i]);
		if (pDataArray[i] != NULL)
		{
			free(pDataArray[i]);
			pDataArray[i] = NULL;    // Ensure address is not reused.
		}
	}
#endif
	return 0;
}

DWORD WINAPI MyThreadFunction(LPVOID lpParam)
{
	PMYDATA pDataArray;
	pDataArray = (PMYDATA)lpParam;
	Sleep(10000);
	printf("Parameters = %d, %d\n", pDataArray->val1, pDataArray->val2);
	return 0;
}
```

**分析程序源代码**

```c
#include <windows.h>
#include <tlhelp32.h> //tool help
#include <tchar.h>
#include <stdio.h>
#include <stdlib.h>

//  Forward declarations:
BOOL GetProcessList();
BOOL ListProcessModules(DWORD dwPID);
BOOL ListProcessThreads(DWORD dwOwnerPID);
void printError(TCHAR* msg);

int main(void)
{
	GetProcessList();
	Sleep(100);
	return 0;
}

BOOL GetProcessList()
{
	HANDLE hProcessSnap;
	HANDLE hProcess;
	PROCESSENTRY32 pe32;
	DWORD dwPriorityClass;

	// Take a snapshot of all processes in the system.
	hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	if (hProcessSnap == INVALID_HANDLE_VALUE)
	{
		printError(TEXT("CreateToolhelp32Snapshot (of processes)"));
		return(FALSE);
	}

	// Set the size of the structure before using it.
	pe32.dwSize = sizeof(PROCESSENTRY32);

	// Retrieve information about the first process,
	// and exit if unsuccessful
	if (!Process32First(hProcessSnap, &pe32))
	{
		printError(TEXT("Process32First")); // show cause of failure
		CloseHandle(hProcessSnap);          // clean the snapshot object
		return(FALSE);
	}

	// Now walk the snapshot of processes, and
	// display information about each process in turn
	do
	{
		_tprintf(TEXT("\n%d\t%s"), pe32.th32ProcessID, pe32.szExeFile);

        // 只有当进程名为 CreateThread.exe 才对其进行分析
		if (lstrcmp(pe32.szExeFile, TEXT("CreateThread.exe")) == 0)
		{
			// Retrieve the priority class.
			dwPriorityClass = 0;
			hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pe32.th32ProcessID);
			if (hProcess == NULL)
				printError(TEXT("OpenProcess"));
			else
			{
				dwPriorityClass = GetPriorityClass(hProcess);
				if (!dwPriorityClass)
					printError(TEXT("GetPriorityClass"));
				CloseHandle(hProcess);
			}

			_tprintf(TEXT("\n  Process ID        = 0x%08X"), pe32.th32ProcessID);
			_tprintf(TEXT("\n  Thread count      = %d"), pe32.cntThreads);
			_tprintf(TEXT("\n  Parent process ID = 0x%08X"), pe32.th32ParentProcessID);
			_tprintf(TEXT("\n  Priority base     = %d"), pe32.pcPriClassBase);
			if (dwPriorityClass)
				_tprintf(TEXT("\n  Priority class    = %d"), dwPriorityClass);

			// List the modules and threads associated with this process
			ListProcessModules(pe32.th32ProcessID);
			ListProcessThreads(pe32.th32ProcessID);
		}

	} while (Process32Next(hProcessSnap, &pe32)); //列表遍历

	CloseHandle(hProcessSnap);
	return(TRUE);
}


BOOL ListProcessModules(DWORD dwPID)
{
	HANDLE hModuleSnap = INVALID_HANDLE_VALUE;
	MODULEENTRY32 me32;

	// Take a snapshot of all modules in the specified process.
	hModuleSnap = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, dwPID);
	if (hModuleSnap == INVALID_HANDLE_VALUE)
	{
		printError(TEXT("CreateToolhelp32Snapshot (of modules)"));
		return(FALSE);
	}

	// Set the size of the structure before using it.
	me32.dwSize = sizeof(MODULEENTRY32);

	// Retrieve information about the first module,
	// and exit if unsuccessful
	if (!Module32First(hModuleSnap, &me32))
	{
		printError(TEXT("Module32First"));  // show cause of failure
		CloseHandle(hModuleSnap);           // clean the snapshot object
		return(FALSE);
	}

	// Now walk the module list of the process,
	// and display information about each module
	do
	{
		_tprintf(TEXT("\n\n     MODULE NAME:     %s"), me32.szModule);
		_tprintf(TEXT("\n     Executable     = %s"), me32.szExePath);
		_tprintf(TEXT("\n     Process ID     = 0x%08X"), me32.th32ProcessID);
		_tprintf(TEXT("\n     Ref count (g)  = 0x%04X"), me32.GlblcntUsage);
		_tprintf(TEXT("\n     Ref count (p)  = 0x%04X"), me32.ProccntUsage);
		_tprintf(TEXT("\n     Base address   = 0x%08X"), (DWORD)me32.modBaseAddr);
		_tprintf(TEXT("\n     Base size      = %d"), me32.modBaseSize);

	} while (Module32Next(hModuleSnap, &me32));

	CloseHandle(hModuleSnap);
	return(TRUE);
}

BOOL ListProcessThreads(DWORD dwOwnerPID)
{
	HANDLE hThreadSnap = INVALID_HANDLE_VALUE;
	THREADENTRY32 te32;

	// Take a snapshot of all running threads
	hThreadSnap = CreateToolhelp32Snapshot(TH32CS_SNAPTHREAD, 0);
	if (hThreadSnap == INVALID_HANDLE_VALUE)
		return(FALSE);

	// Fill in the size of the structure before using it. 
	te32.dwSize = sizeof(THREADENTRY32);

	// Retrieve information about the first thread,
	// and exit if unsuccessful
	if (!Thread32First(hThreadSnap, &te32))
	{
		printError(TEXT("Thread32First")); // show cause of failure
		CloseHandle(hThreadSnap);          // clean the snapshot object
		return(FALSE);
	}

	// Now walk the thread list of the system,
	// and display information about each thread
	// associated with the specified process
	do
	{
		if (te32.th32OwnerProcessID == dwOwnerPID)
		{
			_tprintf(TEXT("\n\n     THREAD ID      = 0x%08X"), te32.th32ThreadID);
			_tprintf(TEXT("\n     Base priority  = %d"), te32.tpBasePri);
			_tprintf(TEXT("\n     Delta priority = %d"), te32.tpDeltaPri);
			_tprintf(TEXT("\n"));
		}
	} while (Thread32Next(hThreadSnap, &te32));

	CloseHandle(hThreadSnap);
	return(TRUE);
}

void printError(TCHAR* msg)
{
	DWORD eNum;
	TCHAR sysMsg[256];
	TCHAR* p;

	eNum = GetLastError();
	FormatMessage(FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
		NULL, eNum,
		MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), // Default language
		sysMsg, 256, NULL);

	// Trim the end of the line and terminate it with a null
	p = sysMsg;
	while ((*p > 31) || (*p == 9))
		++p;
	do { *p-- = 0; } while ((p >= sysMsg) &&
		((*p == '.') || (*p < 33)));

	// Display the message
	_tprintf(TEXT("\n  WARNING: %s failed with error %d (%s)"), msg, eNum, sysMsg);
}
```

**分析结果**

```bash
16560   CreateThread.exe
  Process ID        = 0x000040B0
  Thread count      = 1
  Parent process ID = 0x00000DC0
  Priority base     = 8
  Priority class    = 32

     MODULE NAME:     CreateThread.exe
     Executable     = F:\软件项目开发生命周期\CreateThread\Debug\CreateThread.exe
     Process ID     = 0x000040B0
     Ref count (g)  = 0xFFFF
     Ref count (p)  = 0xFFFF
     Base address   = 0x00DE0000
     Base size      = 131072

     MODULE NAME:     ntdll.dll
     Executable     = C:\WINDOWS\SYSTEM32\ntdll.dll
     Process ID     = 0x000040B0
     Ref count (g)  = 0xFFFF
     Ref count (p)  = 0xFFFF
     Base address   = 0x77730000
     Base size      = 1626112

     MODULE NAME:     KERNEL32.DLL
     Executable     = C:\WINDOWS\System32\KERNEL32.DLL
     Process ID     = 0x000040B0
     Ref count (g)  = 0xFFFF
     Ref count (p)  = 0xFFFF
     Base address   = 0x74760000
     Base size      = 851968

     MODULE NAME:     KERNELBASE.dll
     Executable     = C:\WINDOWS\System32\KERNELBASE.dll
     Process ID     = 0x000040B0
     Ref count (g)  = 0xFFFF
     Ref count (p)  = 0xFFFF
     Base address   = 0x77120000
     Base size      = 1929216

     MODULE NAME:     VCRUNTIME140D.dll
     Executable     = C:\WINDOWS\SYSTEM32\VCRUNTIME140D.dll
     Process ID     = 0x000040B0
     Ref count (g)  = 0x0001
     Ref count (p)  = 0x0001
     Base address   = 0x0F5D0000
     Base size      = 110592

     MODULE NAME:     ucrtbased.dll
     Executable     = C:\WINDOWS\SYSTEM32\ucrtbased.dll
     Process ID     = 0x000040B0
     Ref count (g)  = 0x0002
     Ref count (p)  = 0x0002
     Base address   = 0x0FAE0000
     Base size      = 1523712

     THREAD ID      = 0x0000356C
     Base priority  = 8
     Delta priority = 0
```
