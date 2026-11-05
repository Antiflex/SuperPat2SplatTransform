# Conversion de transformations SuperSplat vers SplatTransform

Ce document décrit la conversion utilisée pour appliquer dans
**SplatTransform** un déplacement/une rotation correspondant au
changement effectué dans **SuperSplat**, puis pour traiter tous les
fichiers `.sog` d'un dossier.

## 1. Convention observée

À l'import d'un `.sog`, SuperSplat applique une rotation initiale de
**180° autour de Z** pour adapter l'orientation du modèle.

Cette rotation initiale ne fait **pas** partie du changement de rotation
demandé ensuite dans SuperSplat. Il ne faut donc pas la réappliquer lors
de la conversion.

Les essais effectués donnent les correspondances suivantes pour le
**delta de rotation** :

  Axe     | SuperSplat  | SplatTransform
  --------|-------------|---------------
  X       |      +90°   |          -90°
  Y       |      +90°   |          -90°
  Z       |      +90°   |          +90°

La règle empirique retenue est donc :

``` text
rotation_ST = (-rotation_SS_X, -rotation_SS_Y, rotation_SS_Z)
```

La translation testée est identique dans les deux logiciels :

``` text
translation_ST = translation_SS
```

### Exemple de rotation combinée

Une rotation demandée dans SuperSplat de `[30, 45, 20]` correspond à
`[-30, -45, 20]` dans SplatTransform.

SuperSplat peut afficher une rotation finale de `[30, 45, -160]` à cause
de la rotation initiale de 180° autour de Z. Pour la conversion, on
utilise bien le delta demandé `[30, 45, 20]`, et non cette rotation
finale.

Autre exemple :

``` text
Rotation finale affichée dans SuperSplat : [-12, 10, -175]
Rotation initiale :                       [0, 0, 180]
Delta demandé (approximatif) :             [-12, 10, 5]
Rotation à passer à SplatTransform :       [12, -10, 5]
```

Les angles sont périodiques : `185°` et `-175°` représentent la même
orientation.

> **Limite importante :** cette conversion par composantes a été déduite
> des essais effectués. Elle n'est pas une preuve mathématique de
> compatibilité pour toutes les rotations possibles. Les conventions
> d'Euler et l'ordre de composition peuvent créer des différences pour
> certaines rotations combinées. Pour une conversion générale garantie,
> il faudrait comparer les matrices/quaternions des deux logiciels.

## 2. Commande SplatTransform utilisée

###  ⚠️ Avant d'entrer les commandes il faut un répertoire avec un environnement node.js et SplatTransform

- assurez-vous d'avoir nodejs installé (ou équivalent) [https://nodejs.org/fr/download](https://nodejs.org/fr/download)
- initialisé npm avec la commande `npm init` dans un dossier (un dossier vide de préférence, cela va créer un package.json, package-lock.json et un dossier /node-modules, prerparer un dossier à l'occasion permet de ne pas polluer les reste de vos fichiers)  
exemple :
1. Le dossier de travail est `D:/source/repos/SuperSplat2SplatTransform/`
2. Dans une fenêtre cmd ou powershell executer les commandes : 
    - `cd D:/source/repos/SuperSplat2SplatTransform`, remplacer 'D:/source/repos/SuperSplat2SplatTransform/' par le chemin vers votre dossier
    - `npm init` 
    - `npm install -g @playcanvas/splat-transform`
    - vous pouvez maintenant utiliser la commande splat-transform (voir [https://github.com/playcanvas/splat-transform#readme](https://github.com/playcanvas/splat-transform#readme) pour l'utilisation ou bien lire la suite)
3. 

La commande correspondante à une rotation de `(-12,10,5)` suivie d'une translation de `(0,3.8,0)` est donc :

``` cmd
splat-transform "path/to/input/dir/frame0.sog" -r '12,-10,5' -t '0,3.8,0' "path/to/output/dir/frame0_out.sog" -w
```

Elle applique successivement :

1.  La transformation principale pour redresser le splat : rotation `(-12, 10, 5)`.
2.  La transformation pour recentrer le modèle : translation `(0, 3.8, 0)`.


## 3. Traiter tous les fichiers `.sog` avec PowerShell

Le script ci-dessous applique exactement les paramètres de la commande
de test à chaque fichier `.sog` directement présent dans le dossier
source.

Il écrit les résultats dans un sous-dossier `transformed` et conserve
les noms des fichiers. Les originaux ne sont pas modifiés.

``` powershell
$inputDir = "Path/to/input/dir/" # chemin dossier contenant les .sog d'entrée
$outputDir = Join-Path $inputDir "transformed"

New-Item -ItemType Directory -Force -Path $outputDir | Out-Null

Get-ChildItem -Path $inputDir -Filter "*.sog" -File | ForEach-Object {
    $input = $_.FullName
    $output = Join-Path $outputDir $_.Name

    Write-Host "Transforming $($_.Name)..."

    & splat-transform $input `
        -r "12,10,5" `
        $output `      # Rotation et translation à remplacer
        -t "0,3.8,0" `
        -w

    if ($LASTEXITCODE -ne 0) {
        Write-Host "ERROR: $($_.Name)" -ForegroundColor Red
    }
}
```

### Exécution

1.  Vérifier que `splat-transform` est installé et accessible depuis
    PowerShell.
2.  Modifier `$inputDir` si les fichiers se trouvent dans un autre
    dossier.
3.  Copier-coller le script dans PowerShell.
4.  Vérifier les fichiers générés dans le sous-dossier `transformed`.

**Important :** les vecteurs sont entourés de guillemets (`"0,3.76,0"`).
Sans ces guillemets, PowerShell peut transmettre les trois coordonnées
comme des arguments séparés, ce qui provoque l'erreur
`Invalid number value: 0 3.8 0`.

Le script ne parcourt pas les sous-dossiers. Il traite uniquement les
`.sog` situés directement dans `$inputDir`.
