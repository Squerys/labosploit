# labosploit
Exploitation de la vérification des réponses côté client pour obtenir une réponse juste peu importe l'entrée

DISCLAIMER : Je n'encourage aucunement à la triche, c'est uniquement une proof of concept sur pourquoi il est important d'intégrer des systèmes de vérifications côté serveur et restreindre certains chemin absolus

Explication:
 - Le point critique de la vérification de labomep est qu'elle est faite par un script chargé côté client uniquement
 - On peut donc logiquement le falsifier en utilisant l'option des remplacements locaux

Lorsque l'on charge une séance labomep:
- Le client récupère les différentes ressources dont il a besoin (textes, assets, images, mais surtout script js de la séance)

<br>En analysant les différents script chargés, on observe un fichier nommé "textes-CnJyC1bT.js" contenant un dictionnaire qui contient les strings à afficher selon si la réponse est juste, incomplète ou exacte. </br>
Notre objectif est de maintenant trouver quand et comment cette liste est chargée pour retrouver la condition de vérification.
- On y trouve une référence dans un autre script qui semble obfusqué légèrement et peu intuitif nommé au chemin absolu /build nommé "sectionsquelettemtg32_[inserer_nom_de_seance]-[caractères_aleatoire].js"
- Par chance, il y a aussi un autre repertoire nommé /src avec un meme fichier "sectionsquelettemtg32_[inserer_nom_de_seance]-Edit-[caractères_aleatoire].js" qui lui n'est pas utilisé par la séance
- Mais qui est quand meme chargé (je ne sais pas pourquoi), c'est une version "squelette" du scripts obfusqué.
- On voit plusieurs reférences à la validation
```
function afficheReponse (bilan, depasse) {
    let ch, coul
    if ((bilan === 'exact')) {
      ch = 'Exact : '
      coul = parcours.styles.cbien
    } else {
      if (bilan === 'correct') {
        ch = 'Vérifié par les ' + sq.nomSolutions + ' : '
        coul = depasse ? parcours.styles.cfaux : '#0000FF'
      } else {
        if (bilan === 'faux') {
          ch = 'Faux : '
          coul = parcours.styles.cfaux
        } else { // exact pas fini
          ch = 'Exact pas fini : '
          coul = depasse ? parcours.styles.cfaux : '#0000FF'
        }
      }
    }
```
Il nous faut maitenant chercher ou est definit le contenu de la variable bilan, on cherche encore un peu
```
case 'correction': {
      let bilanReponse = ''
      let simplificationPossible = false
      if (!sq.nbSolEntre) {
        sq.fcts_valid.validationGlobale()
        const rep = j3pElement('consigneNbSolliste1').selectedIndex - 1
        if (rep === -1) {
          bilanReponse = 'nbSolPasEntre'
        } else {
          if (rep === ((sq.nbSol >= 1) ? 1 : 0)) {
            if (rep === 0) bilanReponse = 'nbSolExactFini' // Cas où il n'y a pas de solution
            else bilanReponse = 'nbSolExact'
          } else {
            bilanReponse = 'nbSolFaux'
          }
        }
      } else {
        // On regarde d'abord l'éditeur a un contenu vide ou incorrect
        if (validationEditeur()) {
          validation()
          if (sq.simplifier) {
            if (sq.resolu) {
              bilanReponse = 'exact'
            } else {
              if (sq.exact) {
                bilanReponse = 'exactPasFini'
                simplificationPossible = true
              } else {
                if (sq.correct) bilanReponse = 'correct'
                else bilanReponse = 'faux'
              }
            }
          } else {
            if (sq.resolu || sq.exact) {
              bilanReponse = 'exact'
              if (!sq.resolu) simplificationPossible = true
            } else {
              if (sq.correct) bilanReponse = 'correct'
              else bilanReponse = 'faux'
            }
          }
        } else {
          bilanReponse = 'incorrect'
        }
      }
```
Bingo! Il faut maintenant retrouver cette structure dans le fichier obfusqué
Et encore une fois bingo,
```
            let t = ""
              , o = !1;
            if (e.nbSolEntre)
                k() ? (B(),
                e.simplifier ? e.resolu ? t = "exact" : e.exact ? (t = "exactPasFini",
                o = !0) : e.correct ? t = "correct" : t = "faux" : e.resolu || e.exact ? (t = "exact",
                e.resolu || (o = !0)) : e.correct ? t = "correct" : t = "faux") : t = "incorrect";
            else {
                e.fcts_valid.validationGlobale();
                const s = n("consigneNbSolliste1").selectedIndex - 1;
                s === -1 ? t = "nbSolPasEntre" : s === (e.nbSol >= 1 ? 1 : 0) ? s === 0 ? t = "nbSolExactFini" : t = "nbSolExact" : t = "nbSolFaux"
            }
```
Bien que ce soit moins lisible, on reconnait clairement la structure, on spécifie avec les outils de développements de charger une version enregistrée localement du script, on change les actions après les conditions pour que peu importe les entrées, la valeur est toujours passé à exact
```
            if (e.nbSolEntre)
                k() ? (B(),
                e.simplifier ? e.resolu ? t = "exact" : e.exact ? (t = "exact",
                o = !0) : e.correct ? t = "exact" : t = "exact" : e.resolu || e.exact ? (t = "exact",
                e.resolu || (o = !0)) : e.correct ? t = "exact" : t = "exact") : t = "exact";
            else {
                e.fcts_valid.validationGlobale();
                const s = n("consigneNbSolliste1").selectedIndex - 1;
                s === -1 ? t = "nbSolExact" : s === (e.nbSol >= 1 ? 1 : 0) ? s === 0 ? t = "nbSolExact" : t = "nbSolExact" : t = "nbSolExact"
            }
```
On sauvegarde puis recharge la séance, et voilà, le système de verification des réponses est complètement cassé, il nous donne juste peu importe l'entrée utilisateur
