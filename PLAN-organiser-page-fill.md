# Plan : Organiser — remplissage de page plutôt que remplissage de bloc

## Constat

Aujourd'hui, le bouton "Organiser" (`organizeBlocks()` dans `docs/index.html`,
~ligne 1589) ajuste chaque bloc de texte **indépendamment** : il mesure la
hauteur naturelle du contenu à `DEFAULT_BODY_FONT_SIZE` (11.5px), en déduit un
`rowSpan` (avec un peu de marge via `ORGANIZE_PAD_PX`), puis cherche la police
la plus grande qui tient dans cette boîte-là, entre `MIN_BODY_FONT_SIZE` (9px)
et `MAX_BODY_FONT_SIZE` (13px), par pas de `FONT_STEP` (0.5px).

Résultat : un bloc court se retrouve avec une police proche du minimum (juste
assez grande pour remplir sa propre petite boîte), même s'il reste beaucoup
d'espace libre plus bas sur la page A4. Le système "juge" bloc par bloc, pas
à l'échelle de la page — d'où l'impression que le rendu est souvent plus
petit que nécessaire.

## Objectif

Faire juger l'Organiser **au niveau de la page entière** : une fois chaque
bloc ajusté à son contenu (comportement actuel, inchangé), regarder l'espace
vertical encore libre sur les 32 lignes de la grille (`GRID_ROWS`) et, s'il y
en a, remonter la police de **tous les blocs de texte ensemble** jusqu'à
remplir la page sans rien faire déborder ni se chevaucher.

## Algorithme proposé

1. **Passe 1 (existante, inchangée)** : pour chaque bloc `kind==="text"`,
   calculer `rowSpan` et `fontSize` comme aujourd'hui (lignes ~1594-1636).
   Contact et photo restent exemptés, comme actuellement.
2. **Repack existant, inchangé** : `resolveCollisions("contact-block")` puis
   `applyGridPositions()` (lignes ~1639-1641).
3. **Nouvelle passe 2 — remplissage de page** : tant que
   - il reste de la marge (voir "Mesurer l'espace libre" ci-dessous), et
   - au moins un bloc de texte n'a pas encore atteint `MAX_BODY_FONT_SIZE`,

   augmenter le `fontSize` de **tous** les blocs de texte de `FONT_STEP`,
   recalculer le `rowSpan` de chacun à cette nouvelle taille (même mesure
   `body.scrollHeight` que la passe 1), relancer
   `resolveCollisions`/`applyGridPositions`, puis vérifier :
   - qu'aucun bloc ne dépasse la grille (`row + rowSpan - 1 <= GRID_ROWS`
     pour tous les blocs affichés, cf. `layoutBlocks()`), et
   - qu'il ne reste aucun chevauchement résiduel (même check que la fin de
     `organizeBlocks()` actuelle, via `rectsOverlap()`).

   Si l'itération casse l'une de ces deux conditions, **annuler cette
   itération** (revenir aux `rowSpan`/`fontSize`/positions de l'itération
   précédente, la dernière connue bonne) et arrêter la boucle — la page était
   déjà pleine.
4. Sortir de la boucle aussi si `MAX_BODY_FONT_SIZE` est atteint partout, ou
   après un nombre d'itérations borné (`(MAX_BODY_FONT_SIZE -
   MIN_BODY_FONT_SIZE) / FONT_STEP`, soit 8 aujourd'hui) pour éviter toute
   boucle infinie en cas de bug.
5. Toast final : garder le message actuel ("Blocs organisés : taille et
   police ajustées à leur contenu.") si tout est propre, ou le toast
   d'avertissement existant si un chevauchement résiduel subsiste malgré
   tout (cas déjà géré aujourd'hui, à ne pas casser).

### Mesurer l'espace libre

Le plus simple : après la passe 1 + repack, calculer la ligne de grille la
plus basse occupée (`max(b.row + b.rowSpan - 1)` sur tous les blocs affichés
par `layoutBlocks()`). S'il reste au moins, disons, 1 ligne de marge avant
`GRID_ROWS` (32), il y a de la place → lancer la passe 2. Alternative plus
fine : comparer la somme des hauteurs naturelles mesurées à la hauteur totale
de la grille, mais le calcul par ligne occupée est plus simple et cohérent
avec le reste du fichier (qui raisonne déjà en lignes de grille partout).

### Question ouverte à trancher avant de coder

Aujourd'hui chaque bloc peut finir avec une police **différente** (un bloc
dense reste petit, un bloc court grossit). Deux options pour la passe 2 :

- **(A) Incrément uniforme** : chaque bloc grossit de `FONT_STEP` à son
  propre rythme, en gardant les écarts relatifs entre blocs (un bloc déjà
  proche du max arrêtera de grossir avant les autres). Le plus proche du
  fonctionnement actuel, le moins de code à changer.
- **(B) Taille commune** : convergence vers **une seule** taille de police
  pour tous les blocs de texte (le plus petit dénominateur commun qui tient
  partout), pour un rendu plus uniforme visuellement — mais ça peut réduire
  la taille de blocs qui avaient de la place d'être encore plus grands
  individuellement.

Recommandation : commencer par (A) (plus simple, cohérent avec l'existant),
et ne passer à (B) que si le rendu (A) semble trop hétérogène à l'usage réel.

## Ce qui ne change pas

- Le bouton reste "à la demande" (clic, pas d'ajustement en direct pendant
  la frappe) — cf. AGENTS.md.
- Contact et photo restent exemptés (comme aujourd'hui).
- `MIN_BODY_FONT_SIZE` / `MAX_BODY_FONT_SIZE` / `FONT_STEP` restent les
  bornes — la passe 2 ne fait que pousser plus souvent vers le plafond
  existant, elle ne le change pas. (Si après coup le plafond de 13px reste
  trop petit une fois la passe 2 en place, le remonter séparément.)
- Le check de chevauchement résiduel + toast d'avertissement existants
  restent la garde-fou final, inchangés dans leur logique.

## Où toucher dans `docs/index.html`

- `organizeBlocks()` (~ligne 1589) : ajouter la passe 2 après le repack de
  la passe 1 actuelle (~ligne 1641), avant le check de chevauchement final
  (~ligne 1644).
- Pas de nouvelle constante requise a priori, sauf peut-être un
  `PAGE_FILL_MARGIN_ROWS` (marge minimale à laisser libre avant de considérer
  qu'il "reste de la place") si un simple "0 ligne libre" s'avère trop juste
  à l'usage.

## Vérification avant de livrer

Suivre l'approche déjà documentée dans AGENTS.md (chromium headless
`--print-to-pdf` + rasterisation, pas de confiance sur simple lecture de
code) :

1. CV court (peu de contenu par bloc) → vérifier que la police grossit
   visiblement plus qu'aujourd'hui, sans dépasser la page A4.
2. CV déjà dense (comme le contenu par défaut de ce projet) → vérifier
   qu'il ne se passe presque rien (peu ou pas de marge disponible), pas de
   régression de débordement.
3. Un CV avec un bloc quasi plein et un bloc très court côte à côte →
   vérifier qu'aucun chevauchement n'apparaît pendant les itérations de la
   passe 2.
4. Les deux templates (Blueprint et Classique) — la passe 2 ne doit rien
   changer au choix du template, seulement aux `fontSize`/`rowSpan` stockés.
5. Réappuyer sur "Organiser" plusieurs fois de suite sur le même CV : doit
   converger (idempotent), pas grossir indéfiniment à chaque clic.
