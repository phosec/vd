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

Lorsqu'une cible est visible sur l'écran d'un joueur dans CSGO, le radar affiche ce joueur à l'aide d'un point rouge. Le client du jeu passe un attribut de type boolean à TRUE que l'on appelera "spotted" dans cet article.
![](2021-08-15-13-15-03.png#center)

Le but est donc de forcer ce mécanisme de manière artificielle afin de gagner un avantage sur les autres joueurs. Nous allons avoir besoin de trouver la structure en mémoire qui corresponds aux joueurs et forcer le passage de ce boolean à TRUE en écrivant directement dans la mémoire du client.

![](2021-08-15-13-23-19.png#center)

Ce radarhack sera de type "internal", cela consiste à injecter un DLL dans le client du jeu pour que notre code ait un accès direct à la mémoire de celui ci. Il nous suffit donc de trouver la bonne adresse mémoire et de passer la variable à TRUE.

## Pré-requis

Les outils utilisés dans cet article sont :
- [Cheat Engine](https://www.cheatengine.org/)
- [Reclass.net](https://github.com/ReClassNET/ReClass.NET/releases/)
- Visual Studio Community avec les additions C++, MSVC 142 et Windows 10 SDK
- Counter-Strike Global Offensive (gratuit sur Steam)

## 1ère étape - Récuperer la liste des joueurs

### Trouver en mémoire le boolean "spotted"

Lancer CSGO avec le paramètre `--insecure` et demarrer une partie local avec des bots. Pour simplifier le process, activez la console dans les options du jeu et executer cette commande une fois en jeu: 
```
mp_roundtime 9999;mp_startmoney 16000;mp_roundtime_defuse 600000;sv_cheats 1;bot_stop 1;mp_restartgame 1;mp_limitteams 30;mp_autoteambalance 0;mp_buytime 99999
```

Une fois la partie demarree, lancez Cheat Engine et attacher l'outil a csgo.exe :

![](2021-08-15-16-36-47.png#center)

Positionnez vous maintenant dans le jeu de maniere a pouvoir faire apparaitre et disparaitre le point rouge du radar facilement puis trouver les adresses grace a CE de la même manière que dans cette video :

{{< vimeo id="587580154" >}}

Dans mon cas, j'ai pu identifier 2 adresses mémoires qui correspondent à la valeur "spotted" de 2 joueurs différents:

![](2021-08-15-17-56-20.png#center)

Nous pouvons vérifier que ce sont les bonnes adresses en changeant leurs valeurs à 1, les points rouges doivent s'afficher sur le radar. Ces adresses sont dynamiques. Elles seront différentes à chaque lancement du jeu. 


### Analyse de la structure du joueur en mémoire

Afin que notre hack fonctionne après chaque redémarrage du jeu, nous devons trouver le pointeur static qui pointe vers la structure du joueur.

Pour cela, faire un clique droit sur l'une des deux adresses mémoires et cliquer sur "Find out what accesses this address" :

![](2021-08-15-17-58-53.png#center)

![](2021-08-15-17-59-40.png#center)

Voici la ligne exacte qui accede a notre variable :

```asm
test [esi+edx*4+0x980], eax
```

Ici EDX=0, donc on peut simplifier l'instruction a esi+0x980, ce qui veut dire que esi contient l'adresse vers la structure du joueur et 0x980 est l'offset du boolean que l'on va a present appeler "spotted".

On a donc donc: 

```
Adresse dynamique de la structure du joueur: 68917730
Offset de spotted: 0x980
Adresse dynamique de spotted = 68917730+0x180=689180B0
```

Pour trouver son pointeur static, nous pouvons chercher avec CE quelle adresse mémoire pointe vers l'adresse dynamique de la structure (68917730) :

![](2021-08-15-18-06-51.png#center)

L'adresse en verte est notre pointeur static, ici : `<client.dll>+4DA31EC` ce qui se traduit par adresse du module client.dll + 4DA31EC.

A l'aide de l'outil Reclass.net, nous allons verifier si cette adresse correspond bien a un joueur et si nous sommes bien a l'interieur d'une liste.

Fermer CE (ou dettacher le debugger CE en re-attachant CE a csgo.exe), puis ouvrir Reclass et attacher son debugger a csgo.exe. 

Une fois fait, double cliquer sur l'adresse initiale a la premiere ligne et ecrire le module puis l'offset de la structure du joueur, dans notre cas `<client.dll>+4DA31EC` (ne pas oublier les `<>` autours du module) :

![](2021-08-15-18-17-39.png#center)

La premiere ligne corresponds a un pointeur vers la HEAP (le joueur sur lequel nous avons fait le test avec CE), puis d'autre pointeur vers la HEAP qui correspondent a d'autre joueurs. Nous sommes bien dans une liste de joueur.

L'ecart en mémoire entre chaqu'un de ces pointeurs est de `0x10`, information qui nous servira a iterer sur chaque joueur plus tard.

Pour trouver le debut de cette liste, nous pouvons changer l'adresse de depart dans Reclass a `<client.dll>+4DA3100` puis faire clique droit sur la premiere ligne et `Add 2048 Bytes` :

![](2021-08-15-18-23-06.png#center)

En scrollant vers le bas, on retombe sur la liste de joueur, ce qui nous permet de recuperer l'offset exacte du joueur 1 :

![](2021-08-15-18-27-08.png#center)

L'adresse du 1er joueur de la liste est donc `<client.dll>+4DA31BC` (0x00BC : premiere colonne de la ligne du pointeur du joueur 1), pour resumer, nous avons pu recuperer ces informations :

```
Offset de spotted: 0x980
Adresse static du pointeur vers le 1er joueur de la liste: <client.dll>+4DA31BC
Offset entre chaque joueur dans la liste 0x10
```

Il nous suffit donc de creer une DLL, qui une fois injectee dans csgo.exe va resoudre l'adresse du module "client.dll", y ajouter l'offset 4DA31BC puis boucler sur chaque joueur en ajoutant 0x10 a cet offset. 

A chaque iteration, nous pourrons setter le boolean spotted a TRUE qui se trouve a joueur+0x980.


## 2ème étape - Creation de la DLL

Creer un nouveau projet DLL en C++:

![](2021-08-15-18-40-37.png#center)
![](2021-08-15-18-41-13.png#center)

Nous allons commencer par creer un nouveau Thread lors de l'attachement de notre DLL :

```cpp
    case DLL_PROCESS_ATTACH: {
        HANDLE hThread = nullptr;
        hThread = CreateThread(nullptr, 0, (LPTHREAD_START_ROUTINE)RadarHackThread, hModule, 0, nullptr);
        if (hThread)
            CloseHandle(hThread);
    }
```

Puis declarer notre Thread comme ceci:

```cpp
DWORD WINAPI RadarHackThread(HMODULE hModule) {
    while (true) {
        if (GetAsyncKeyState(VK_END) & 1) break;
    }
    FreeLibraryAndExitThread(hModule, 0);
    Sleep(100);
    return 0;
}
```

Le seul include necessaire est `<Windows.h>`. Cette base de code permet de creer un Thread puis de le quitter proprement en appuyant sur la touche END du clavier.

On utilise l'API Windows `GetModuleHandle()` pour recuper l'adresse du module "client.dll" puis on declare la liste de joueurs grace a l'offset precedemment trouve :

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

On itere sur les joueurs en incrementant l'adresse de `0x10` puis on met spotted a TRUE :
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


Un injecteur public puissant et gratuit : https://guidedhacking.com/resources/guided-hacking-dll-injector.4

Pour aller plus loin : https://guidedhacking.com/forums/the-game-hacking-bible-learn-how-to-hack-games.469/

