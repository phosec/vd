---
title: "CS:GO Radar Hack en moins de 40 lignes de code"
date: 2021-08-14
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

## Qu'est-ce qu'un RadarHack dans CSGO ?

Le radar affiche les cibles visibles à l'aide de points rouges. Techniquement, le client du jeu passe un attribut de type boolean à TRUE que l'on appelera "spotted" dans cet article.
![](2021-08-15-13-15-03.png#center)

Le but est de forcer ce mécanisme afin de gagner un avantage sur les autres joueurs. En trouvant la structure en mémoire qui corresponds aux joueurs, nous pourrons changer la valeur de ce boolean à TRUE en écrivant dans la mémoire du client.

![](2021-08-15-13-23-19.png#center)

## Pré-requis

Les outils utilisés dans cet article sont :
- [Cheat Engine](https://www.cheatengine.org/)
- [Reclass.net](https://github.com/ReClassNET/ReClass.NET/releases/)
- Visual Studio Community avec les additions C++, MSVC 142 et Windows 10 SDK
- Counter-Strike Global Offensive (gratuit sur Steam)

## 1ère étape - Récuperer la liste des joueurs

### Trouver en mémoire le boolean "spotted"

Lancer CSGO avec le paramètre `--insecure` et demarrer une partie local avec des bots. Pour simplifier le process, activez la console dans les options du jeu et éxécuter cette commande une fois en jeu: 
```
mp_roundtime 9999;mp_startmoney 16000;mp_roundtime_defuse 600000;sv_cheats 1;bot_stop 1;mp_restartgame 1;mp_limitteams 30;mp_autoteambalance 0;mp_buytime 99999
```

Une fois la partie démarrée, lancez Cheat Engine et attacher l'outil à csgo.exe :

![](2021-08-15-16-36-47.png#center)

Positionnez vous dans le jeu de manière à pouvoir faire varier la valeur "spotted" facilement puis identifier leurs adresses mémoires avec CE :

{{< vimeo id="587580154" >}}

Dans mon cas, j'ai pu identifier deux adresses mémoires qui correspondent aux valeurs "spotted" de 2 joueurs différents:

![](2021-08-15-17-56-20.png#center)

En changeant leurs valeurs à 1, les points rouges doivent s'afficher sur le radar. Ces adresses étant dynamiques, elles seront différentes à chaque lancement du jeu. 


### Analyse de la structure du joueur en mémoire

Afin que notre hack fonctionne après chaque redémarrage du jeu, nous devons trouver un pointeur static qui pointe vers la structure du joueur.

Pour cela, faites un clique droit sur l'une des deux adresses mémoires et cliquer sur "Find out what accesses this address" :

![](2021-08-15-17-58-53.png#center)

![](2021-08-15-17-59-40.png#center)

Voici la ligne éxacte qui accède à notre variable :

```asm
test [esi+edx*4+0x980], eax
```

Ici EDX=0, on peut simplifier l'instruction à ESI+0x980, ESI contient l'adresse vers la structure du joueur et 0x980 est l'offset du boolean "spotted".

On à donc : 

```
Adresse dynamique de la structure du joueur: 0x68917730
Offset de spotted: 0x980
Adresse dynamique de spotted = 0x68917730+0x980=0x689180B0
```

Pour trouver son pointeur static, nous pouvons chercher avec CE quelle adresse mémoire pointe vers l'adresse dynamique de la structure (68917730) :

![](2021-08-15-18-06-51.png#center)

Cheat Engine représente les pointeurs statics en vert, ici : `<client.dll>+4DA31EC`, `0x4DA31EC` est l'offset d'un joueur.

Avec l'outil Reclass.net, nous allons vérifier si cette adresse correspond bien à la liste des joueurs et si l'offset que nous avons trouvé corresponds au 1er joueur de cette liste.

Fermer Cheat Engine, puis ouvrir Reclass et attacher son debugger à csgo.exe. 

Une fois fait, double cliquer sur l'adresse initiale à la première ligne et écrire le module puis l'offset du joueur, dans notre cas `<client.dll>+4DA31EC` :

![](2021-08-15-18-17-39.png#center)

Nous sommes bien dans une liste de joueur. La première ligne corresponds a un pointeur vers la HEAP (la structure d'un joueur), puis d'autre pointeur vers la HEAP qui correspondent aux autres joueurs.

L'écart en mémoire entre chaqu'un de ces pointeurs est de `0x10`, cette information nous servira à itérer sur les joueurs.

Pour trouver le début de cette liste, nous pouvons changer l'adresse de départ dans Reclass à `<client.dll>+4DA3100` puis faire clique droit sur la première ligne et `Add 2048 Bytes` :

![](2021-08-15-18-23-06.png#center)

En scrollant vers le bas, on retombe sur la liste de joueur, ce qui nous permet de récupérer l'offset du premier joueur de la liste :

![](2021-08-15-18-27-08.png#center)

L'offset est `<client.dll>+4DA31BC`, pour résumer, nous avons ces informations :

```
Offset de spotted: 0x980
Offset du pointeur correspondant au 1er joueur de la liste: <client.dll>+4DA31BC
Écart entre chaque joueur dans la liste 0x10
```

## 2ème étape - Création de la DLL

Créer un nouveau projet DLL en C++:

![](2021-08-15-18-40-37.png#center)
![](2021-08-15-18-41-13.png#center)

Premièrement, on créé un nouveau Thread lors de l'injection de notre DLL :

```cpp
    case DLL_PROCESS_ATTACH: {
        HANDLE hThread = nullptr;
        hThread = CreateThread(nullptr, 0, (LPTHREAD_START_ROUTINE)RadarHackThread, hModule, 0, nullptr);
        if (hThread)
            CloseHandle(hThread);
    }
```

Puis on déclare notre Thread comme ceci:

```cpp
DWORD WINAPI RadarHackThread(HMODULE hModule) {
    while (true) {
        if (GetAsyncKeyState(VK_END) & 1) break; // La touche END désactive le hack proprement
    }
    FreeLibraryAndExitThread(hModule, 0);
    Sleep(100);
    return 0;
}
```

On déclare la liste de joueurs grace à l'offset précédemment trouvé :

{{< highlight cpp "linenos=table,hl_lines=2 3,linenostart=1" >}}
DWORD WINAPI RadarHackThread(HMODULE hModule) {
    uintptr_t client = (uintptr_t)GetModuleHandle(L"client.dll");
    uintptr_t playerList = (client + 0x4DA31BC);

    while (true) {
        if (GetAsyncKeyState(VK_END) & 1) break;
    }
    FreeLibraryAndExitThread(hModule, 0);
    Sleep(100);
    return 0;
}
{{< / highlight >}}

On itère sur les joueurs en incrémentant l'adresse de `0x10` puis on met spotted a TRUE :

{{< highlight cpp "linenos=table,hl_lines=7-13,linenostart=1" >}}
DWORD WINAPI RadarHackThread(HMODULE hModule) {
    uintptr_t client = (uintptr_t)GetModuleHandle(L"client.dll");
    uintptr_t playerList = (client + 0x4DA31BC);

    while (true) {
        if (GetAsyncKeyState(VK_END) & 1) break;
        for (int i = 0; i < 32; i++) {
            uintptr_t player = *(uintptr_t* )(playerList + 0x10 * i);
            if (player) {
                int* spotted = (int*)(player + 0x980);
                *spotted = TRUE;
            }
        }
    }
    FreeLibraryAndExitThread(hModule, 0);
    Sleep(100);
    return 0;
}
{{< / highlight >}}


En injectant ce DLL dans csgo.exe, le RadarHack devrait etre fonctionnel :

{{< vimeo id="587598459" >}}


Pour créer votre propre injecteur : [Injection de DLL - LoadLibrary]({{< ref "/articles/injection-dll-loadlibrary/index.md" >}})
