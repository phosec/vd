---
title: "Injection de DLL - LoadLibrary"
date: 2021-08-19
# weight: 1
# aliases: ["/first"]
tags: ["csgo", "gamehacking", "reverse"]
author: "phos"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: true
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
---

## L'injection de DLL avec LoadLibrary

Cette article va couvrir la téchnique la plus simple pour injecter un DLL dans un processus distant.

Cette méthode est assez simple a exécuter, il suffit de :
- Etape 1 - Ecrire dans le processus cible le chemin vers votre DLL
- Etape 2 - Exécuter LoadLibrary() dans le processus cible grace à CreateRemoteThread()

## Ecrire dans le processus cible le chemin vers votre DLL

Pour écrire dans un processus distant, nous avons besoin d'obtenir un HANDLE avec `OpenProcess()` :

{{< highlight cpp "linenos=table,hl_lines=4,linenostart=1" >}}
int main() {
    const char* dllFilePath = "C:\\test\\radarhack.dll";
    DWORD PID = 12412;
    HANDLE targetProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
}
{{< / highlight >}}

On alloue une zone mémoire dans le processus cible puis on y écrit le chemin vers notre DLL :

{{< highlight cpp "linenos=table,hl_lines=5-6,linenostart=1" >}}
int main() {
    const char* dllFilePath = "C:\\test\\radarhack.dll";
    DWORD PID = 12412;
    HANDLE targetProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
    LPVOID dllFilePathAddress = VirtualAllocEx(targetProcess, NULL, strlen(dllFilePath), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    WriteProcessMemory(targetProcess, dllFilePathAddress, dllFilePath, strlen(dllFilePath), nullptr);
}
{{< / highlight >}}

## Executer LoadLibrary() dans le processus cible grace a CreateRemoteThread()

Enfin, on récupère l'adresse mémoire de `LoadLibraryA()` dans le module `kernel32.dll`. On l'exécute ensuite avec `CreateRemoteThread()` en passant un argument l'adresse mémoire vers le chemin de notre DLL :
{{< highlight cpp "linenos=table,hl_lines=7-8,linenostart=1" >}}
int main() {
    const char* dllFilePath = "C:\\test\\radarhack.dll";
    DWORD PID = 12412;
    HANDLE targetProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, PID);
    LPVOID dllFilePathAddress = VirtualAllocEx(targetProcess, NULL, strlen(dllFilePath), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
    WriteProcessMemory(targetProcess, dllFilePathAddress, dllFilePath, strlen(dllFilePath), nullptr);
    LPVOID loadLibraryAddress = GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
    CreateRemoteThread(targetProcess, NULL, 0, (LPTHREAD_START_ROUTINE)loadLibraryAddress, dllFilePathAddress, 0, NULL);
}
{{< / highlight >}}

{{< vimeo id="589612343" >}}

En injectant votre DLL avec `LoadLibrary()` vous aurez accès à toutes les fonctionnalités de debugging (breakpoints etc). 

En revanche, si on inspecte la liste des modules chargés par le processus cible, on retrouve notre `radarhack.dll`, ce qui rend cette méthode facilement détectable :

![](2021-08-19-20-30-52.png#center)

La prochain article sur le sujet sera consacré a l'injection de DLL avec Manual Mapping (un peu plus stealthy).