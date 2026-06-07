# Lab 13 – Contournement de la détection root sur Android

**Auteur :** Charaf  
**Application testée :** Uncrackable1 (`owasp.mstg.uncrackable1`)  
**Outils utilisés :** Frida, Objection, ADB

---

## Contexte

Certaines applications Android refusent de fonctionner sur un appareil rooté. Elles vérifient la présence de binaires comme `su`, de packages suspects ou de propriétés système révélatrices. Dans ce lab, j'ai utilisé Objection (basé sur Frida) pour contourner cette protection sur l'application `Uncrackable1` de l'OWASP MSTG.

---

## Mise en place

### Installation d'Objection

J'ai installé Objection via `pipx` depuis le terminal :

```powershell
pipx install objection
```

Une fois l'installation terminée, la commande `objection.exe` est disponible dans le PATH.

<img width="452" height="61" alt="image" src="https://github.com/user-attachments/assets/751b7f97-9fc6-4d4f-8a64-656a4714c116" />


### Vérification avec Frida

Avant de commencer, j'ai vérifié que Frida reconnaissait correctement l'émulateur et l'application cible. J'ai lancé `frida-server` sur l'appareil Android, puis depuis le PC :

```powershell
frida-ps -Uai
```

L'application apparaît bien dans la liste :

```
Uncrackable1    owasp.mstg.uncrackable1
```

<img width="645" height="403" alt="image" src="https://github.com/user-attachments/assets/67e31fd1-498c-4f0e-9932-dff48b0ee028" />


---

## Comportement de l'application sans bypass

Au lancement normal de l'application sur l'émulateur rooté, elle détecte immédiatement l'environnement et affiche :

```
Root detected!
This is unacceptable. The app is now going to exit.
```

L'application se ferme automatiquement. La protection est donc bien active.

<img width="200" height="289" alt="image" src="https://github.com/user-attachments/assets/c3580072-8759-47f5-8edb-e6a8002b82f9" />


---

## Bypass avec Objection

Pour contourner cette protection, j'ai attaché Objection au processus de l'application :

```powershell
objection -g owasp.mstg.uncrackable1 explore
```

Une fois dans la console interactive d'Objection, j'ai exécuté :

```
android root disable
```

Cette commande injecte automatiquement des hooks Frida qui neutralisent les méthodes Java de détection root les plus courantes : recherche de fichiers `su`, vérification de packages, appels à `Runtime.exec`, etc.

La console confirme que le job `root-detection-disable` est bien enregistré et actif.

<img width="597" height="207" alt="image" src="https://github.com/user-attachments/assets/3e5336fe-d0b1-4fe0-9482-00bc554e375f" />


---

## Ce que fait `android root disable`

Derrière cette commande, Objection installe des hooks sur plusieurs points de vérification côté Java :

- **`File.exists()`** → retourne `false` pour les chemins suspects (`/sbin/su`, `/system/xbin/su`, etc.)
- **`Runtime.exec()`** → bloque les commandes `which su`, `busybox`, etc.
- **`PackageManager`** → cache les packages root comme Magisk ou SuperSU
- **`Build.TAGS`** → retourne `release-keys` au lieu de `test-keys`
- **Propriétés système** → `ro.debuggable` forcé à `0`, `ro.secure` forcé à `1`

L'application continue de tourner mais toutes ses vérifications root renvoient des résultats normaux.

---

## Résultat

Après l'activation du module `android root disable`, l'application ne détecte plus le root et démarre normalement. Le bypass est entièrement dynamique : aucune modification de l'APK n'est nécessaire.

---

## Conclusion

Ce lab illustre une limite fondamentale des protections root côté Java : elles peuvent être contournées par instrumentation dynamique. Objection rend cette opération accessible en une seule commande. Pour une protection plus robuste, une application devrait combiner des vérifications natives (NDK) avec des contrôles côté serveur, ce qui rend le bypass beaucoup plus difficile.
