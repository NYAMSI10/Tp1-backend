# TP 1 — Lire les mesures sans bloquer la boucle d'événements

On mesure ce que subit un autre utilisateur (la sonde, qui envoie un `/ping`
toutes les 10 ms) pendant l'import d'un fichier de mesures, d'abord en lecture
bloquante (`readFileSync`), puis en flux (`createReadStream` + `readline`).

## Reproduire les mesures

```bash
# 1. Générer le fichier de 30 jours (144 000 lignes)
node scripts/generate-measures.js SN-CASA 2026-10-31 30

# 2. Terminal A — serveur avec le fichier de 30 jours
node --watch tp/tp1-lecture/serveur-depart.js data/mesures-SN-CASA-2026-10-31_30j.ndjson

# 3. Terminal B — sonde
node tp/tp1-lecture/sonde.js bloquant
node tp/tp1-lecture/sonde.js flux

# 4. Recommencer avec le fichier d'une journée : Ctrl+C dans le terminal A,
#    puis relancer SANS argument, et relancer la sonde dans le terminal B
node --watch tp/tp1-lecture/serveur-depart.js
```

## Relevé des mesures

| Fichier  | Lecture   | Durée de l'import (ms) | Pire temps de /ping (ms) |
|----------|-----------|------------------------|--------------------------|
| 1 jour   | bloquante | 6                      | 2                        |
| 1 jour   | en flux   | 8                      | 2                        |
| 30 jours | bloquante | 99                     | 90                       |
| 30 jours | en flux   | 105                    | 2                        |

> Contrôle : 144 000 mesures et un poids moyen de **38,74 kg** avec le fichier de
> 30 jours ; 4 800 mesures et **37,87 kg** avec celui d'une journée. Seules les
> durées varient d'un poste à l'autre.

## Les trois questions

**1. Pourquoi `/ping` attend-il pendant l'import bloquant ?**
`readFileSync` puis la boucle de `JSON.parse` sur les 144 000 lignes s'exécutent
d'un seul tenant sur le thread principal. Tant que ce travail n'est pas fini, la
boucle d'événements ne peut traiter aucune autre requête : `/ping` reste en file
d'attente et n'est servi qu'une fois l'import terminé (≈ 90 ms de retard).

**2. La lecture en flux est-elle plus rapide ?**
Non — elle est même un peu plus lente (105 ms contre 99 ms sur 30 jours), car
découper le flux en lignes et les livrer une à une a un coût. Ce qu'elle apporte
n'est pas la vitesse mais la *disponibilité* : entre deux morceaux, la boucle
d'événements reprend la main et sert les autres requêtes. Le pire `/ping` tombe
de 90 ms à 2 ms : le serveur reste réactif pendant tout l'import.

**3. Pourquoi l'effet est-il presque invisible avec le fichier d'une journée ?**
L'import d'une journée ne dure que ≈ 6 ms, soit moins qu'un intervalle entre deux
`/ping` (10 ms). Le blocage est trop court pour qu'une requête tombe pile dedans :
qu'on lise en bloquant ou en flux, le pire `/ping` reste à 2 ms. Le problème ne se
voit qu'à partir d'un volume où la durée de l'import dépasse largement ce rythme.
