# Appliquez des filtres sur vos vidéos avec Javascript

Dénicher des images exemptes de retouches relève presque de l'exploit. Et pour cause, lorsque nous ne les manipulons pas nous mêmes, nos appareils photo, téléphonnes, et autres logiciels traitent automatiquement nos images à notre insu, que ce soit pour améliorer la netteté, réhausser la luminosité, améliorer la répartition des couleurs, etc.

S'il convient d'effectuer tout ces traitements en amont pour des raisons d'optimisations, il est parfois nécessaire d'appliquer des filtres sur un contenu vidéo à la volée. Dans ce tutoriel ce tutoriel, nous implémenterons une solution permettant de manipuler le flux de notre image directement depuis notre navigateur.

Nous implémenterons des filtres basiques pour illustrer nos exemples, mais le traitement d'image restera relativement basique.

Maintenant que vous savez à quoi vous attendre, inutile d'attendre plus longtemps : commençons à coder !

## Principe

La méthode présentée dans ce tutoriel consiste à :
- Intégrer une vidéo la notre page, grâce à la balise `<video>` (idéalement en caché)
- Récupérer son flux vidéo dans un objet `ImageData`
- Effectuer le traitement de l'image dessus
- Afficher le résultat dans une balise `<canvas>`.


### Mise en oeuvre


#### Intégrer une vidéo à la page

Rien de bien complexe si vous connaissez un peu le HTML.
```html
 <video
   id="tuto-video"
   src="your-video-url"
   width="300"
   height="300"
   controls
></video>
```


#### Récupération du flux vidéo

Si vous vous êtes déjà intéressés à l'encodage des vidéos, vous savez qu'obtenir le flux de pixels à partir d'un fichier est une tâche relativement complexe. Une bonne connaissance des formats vidéos est indispensable.
Toutefois, grâce à la balise `<video>`, nous somme délivré de cette contrainte. Le navigateur s'occupe d'extraire les pixels, d'effectuer le rendu, et en charge la plupart des formats habituels.

Il ne nous reste qu'à récupérer ce flot de pixel dans notre objet `ImageData`.  Encore une fois, le navigateur va nous rendre un immense service. En effet, l'extraction s'opère en quelques lignes seulement en utilisant un objet `canvas`.

```js
  const video = document.getElementById('tuto-video');

// Create canvas for video's pixel extraction
const extractPixelCanvas = document.createElement('canvas');
const extractPixelContext = extractPixelCanvas.getContext('2d');

/**
 * @return {ImageData}
 */
function extractVideoImageData(video, width, height) {
    // avoid unnecessary resize as much as possible (optimization)
    if (extractPixelCanvas.width !==  width) {
        extractPixelCanvas.width =  width;
    }

    if (extractPixelCanvas.height !==  height) {
        extractPixelCanvas.height = height;
    }
    
    extractPixelContext.drawImage(video, 0, 0, extractPixelCanvas.width, extractPixelCanvas.height);
    return extractPixelContext.getImageData(0, 0, extractPixelCanvas.width, extractPixelCanvas.height);
}
```


#### Manipuler l'objet `ImageData`

Le choix de cette classe est motivé par la facilité d'afficher le contenu d'une instance d'`ImageData` dans un canvas : 
```js
canvasContext2D.putImageData(instanceOfImageData, 0, 0);
```

La structure de l'objet lui même est relativement simple ; une instance possède les propriétés **width** et **height** correspondant à la résolution de l'image, ainsi que **data**, un tableau 8 bits non signés (ou `Uint8ClampedArray`).
Ce dernier attribut nous intéresse plus particulièrement car il contient les pixels de notre image. Chaque pixel est encodé en RGBA, avec un alpha compris entre 0 et 255.

Pour modifier une image à la volée, il suffit de modifier les pixels contenu dans **data**.

Un exemple tiré de [la documentation](https://developer.mozilla.org/en-US/docs/Web/API/ImageData/ImageData).

```js
// Iterate through every pixel
for (let i = 0; i < arr.length; i += 4) {
  arr[i + 0] = 0;    // R value
  arr[i + 1] = 190;  // G value
  arr[i + 2] = 0;    // B value
  arr[i + 3] = 255;  // A value
}
```

Cette classe sera au coeur de l'implémentation de nos filtre, vous trouverez donc quelques exemples supplémentaires par la suite 😉


#### Afficher le résultat dans un `<canvas>`

Rien de bien complexe si vous avez compris les paragraphes précédents.

```html
<!--html-->
<canvas id="tuto-canvas"></canvas>
```
```js
//js
const canvas = document.getElementById('tuto-canvas');
canvasContext2D = canvas.getContext('2d');

const instanceOfImageData = applyYourAmazingFilter(/* ... */);

canvasContext2D.putImageData(instanceOfImageData, 0, 0);
```

## Un filtre, oui ; mais aussi une animation !

L'utilisation d'un filtre sur un flux vidéo est considérée (ici) comme une animation. L'application du filtre à la volée sur des images constitue notre **méthode de rendu**, tandis que la synchronisation entre le canvas et le lecteur vidéo déterminera le comportement de notre **boucle de rendu**. Cette remarque vous paraît floue ? Peut-être devriez-vous jeter un coup d'œil l'article [Faîtes vos propres animations en JS](https://dev.to/qphilippot/faites-vos-propres-animations-en-js-34ok).


### Synchroniser l'animation avec le lecteur vidéo - Définir la boucle de rendu

Nous souhaitons que notre boucle de rendu soit piloté par le lecteur vidéo. C'est à dire que l'animation se lance quand on clique sur play, qu'elle s'arrête lorsqu'on atteint la fin de la vidéo ou que l'on met pause (pour ne pas rafraîchir une image qui ne change pas, ce serait dommage de gâcher des ressources CPU pour rien).

Pour rappel, la **boucle de rendu** s'occupera de rafraîchir automatiquement notre canvas.

```js
const animation = new Animation({ /* … */ });

video.addEventListener('play', () => {
   animation.play();
});

video.addEventListener('pause', () => {
   animation.pause();
});

video.addEventListener('end', () => {
   animation.pause();
});

// render animation once when we click on timeline
video.addEventListener('timeupdate', () => {
   animation.askRendering()
});
```

### Implémentation d'un filtre - Définir une méthode de rendu

Une fois que nous savons comment extraire les pixels de notre vidéo, et que nous avons définit notre boucle de rendu afin que notre canvas se rafraîchisse automatiquement, il ne reste plus qu'à définir notre méthode de rendu.

```js
const animation = new Animation({
    canvas: document.getElementById('tuto-canvas'),
    // rendering method is here
    render: (context, canvas) => {
        const imageData = extractVideoImageData(video, canvas.width, canvas.height);
        // apply filter over imageData here;
        animation.clear();
        context.putImageData(imageData, 0, 0);
        }
    }
);

```



Utilisation classique de canvas et d'image data. Il existe beaucoup de documentation à ce sujet, je vous redirige vers la documentation pour plus d'explication : https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Pixel_manipulation_with_canvas

Note : ce tutoriel montre comment appliquer des filtres sur des images. Il constitue un excellent complément à ce tuto. N'hésitez pas à y jeter un oeil !


## Résumé 

```js
import Animation from '../../shared/animation.model';

document.addEventListener('DOMContentLoaded', () => {
    // Create canvas for video's pixel extraction
    const extractPixelCanvas = document.createElement('canvas');
    const extractPixelContext = extractPixelCanvas.getContext('2d');

    function extractVideoImageData(video, width, height) {
        // avoid unnecessary resize as much as possible (optimization)
        if (extractPixelCanvas.width !==  width) {
            extractPixelCanvas.width =  width;
        }
    
        if (extractPixelCanvas.height !==  height) {
           extractPixelCanvas.height = height;
        }


       extractPixelContext.drawImage(video, 0, 0, extractPixelCanvas.width, extractPixelCanvas.height);
       return extractPixelContext.getImageData(0, 0, extractPixelCanvas.width, extractPixelCanvas.height);
    }
    
    const video = document.getElementById('tuto-video');


    const animation = new Animation({
        canvas: document.getElementById('tuto-canvas'),
        render: (context, canvas) => {
            const imageData = extractVideoImageData(video, canvas.width, canvas.height);
            
            // apply filter over imageData here;
            
           animation.clear();
           context.putImageData(imageData, 0, 0);
       }
   });


    video.addEventListener('play', () => {
        animation.play();
    });

    video.addEventListener('pause', () => {
        animation.pause();
    });

    video.addEventListener('end', () => {
        animation.pause();
    });

    video.addEventListener('timeupdate', () => {
        animation.askRendering()
    })
});
```

## Résultat Préliminaire


![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/no-filter-1.gif?raw=true)

*« - Hein ? Je ne vois aucune différence…»* 🙈


Précisément ! Nous n'avons encore appliqué aucun filtre. Toutefois, on constate que notre flux vidéo est bel et bien répliqué sans déformation ni latence.

Pour appliquer un filtre sur l'image, il suffira d'appliquer un traitement sur l'image data extraire dans la méthode de rendu. Le tutoriel pourrait s'arrêter là, le mécanisme lui n'évoluera pas.

Mais je ne peux pas me résigner à m'arrêter alors que cela commençait tout juste à devenir cool ! 

# Exemple d'implémentation de filtres en JS

## Grayscale

Classique, s'il en est. Nous allons simplement transformer les pixels RGB en niveau de gris.

```js
// get grayscale value for a pixel in buffer

function rgbToGrayscale(buffer, offset) {
   return Math.ceil((
       0.30 * buffer[offset] +
       0.59 * buffer[offset + 1] +
       0.11 * buffer[offset + 2]
   ) * (buffer[offset + 4] / 255.0));
}

/**
* @param {Uint8Array} pixelBuffer
*/
function applyGrayscaleFilter(pixelBuffer) {
   for (let offset = 0; offset <pixelBuffer.length; offset += 4) {
       const grayscale = rgbToGrayscale(pixelBuffer, offset);
       pixelBuffer[offset] = grayscale;
       pixelBuffer[offset + 1] = grayscale;
       pixelBuffer[offset + 2] = grayscale;
       pixelBuffer[offset + 3] = 255;
   }
}


const animation = new Animation({
   canvas: document.getElementById('tuto-canvas'),
   render: (context, canvas) => {
       const imageData = extractVideoImageData(video, canvas.width, canvas.height);
       applyGrayscaleFilter(imageData.data);
       
       animation.clear();
       context.putImageData(imageData, 0, 0);
   }
});
```


Pour chaque pixel, nous remplaçons les canaux RGB par leur niveau de gris.

Intuitivement, il serait tentant d'utiliser une moyenne des composantes RGB. Toutefois, puisque notre oeil ne perçoit pas toutes les couleurs avec la même sensibilité, une telle méthode ne serait pas optimale pour l'œil humain. On utilise à la place la **luminance** du pixel. Ce n'est rien d'autre qu'une combinaison des trois couleurs, mais avec des coefficients pour prendre en compte notre sensibilité à la perception des couleurs.

![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/grayscale.gif?raw=true)

## Supporter les interactions souris

Une animation c'est bien, mais une animation qui interagit avec la souris, c'est mieux ! Nous allons transformer notre code afin d'appliquer le filtre seulement si notre pointeur se trouve à l'intérieur du canvas.

Nous pouvons d'ores et déjà transformer notre méthode de rendu de la sorte :

```js
const animation = new Animation({
    canvas: document.getElementById('tuto-canvas'),
    render: (context, canvas) => {
        const imageData = extractVideoImageData(video, canvas.width, canvas.height);
        
        // compute isPointerHoverCanvas ...

        if (isPointerHoverCanvas === false) {
            applyGrayscaleFilter(imageData.data);
        }
        
        animation.clear();
        context.putImageData(imageData, 0, 0);
    }
});
```

Plusieurs solutions existent pour déterminer si le curseur est, ou n'est pas, à l'intérieur de notre canvas. Selon l'approche, certaines sont plus appropriées que d'autres.

### Déterminer la position de la souris par rapport au canvas

La solution la plus simple consiste à récupérer les coordonnées de la souris, les coordonnées du canvas, et de vérifier si le curseur de la souris est dans la boîte englobante du canvas (ou hitbox).

```js
const pointerCoords = {x: 0, y: 0};

document.addEventListener('pointermove', event => {
    pointerCoords.x = event.clientX;
    pointerCoords.y = event.clientY;
});

const animation = new Animation({
    canvas: document.getElementById('tuto-canvas'),
    render: (context, canvas) => {
        // …

        const boundingBox = canvas.getBoundingClientRect();


        const isPointerHoverCanvas = (
            pointerCoords.x >= boundingBox.left &&
            pointerCoords.y >= boundingBox.top &&
            pointerCoords.x < boundingBox.right &&
            pointerCoords.y < boundingBox.bottom
        );


        if (isPointerHoverCanvas === false) {
            applyGrayscaleFilter(imageData.data);
        }
    }
});
```

![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/grayscale-pointer-1.gif?raw=true)

## On corse le jeu !

On va appliquer le filtre grayscale sur toute l'image, et ne faire apparaître les couleurs que sur les pixels autour de notre curseur.

Petite subtilité : pour créer un effet plus lisse, on déterminera un cercle à l'intérieur duquel les pixels seront colorés, mais avec une intensité inversement proportionnelle à la distance au centre…

### Rappel Géométrique
Un cercle peut être défini par son centre, et un rayon. Dans notre cas, le centre du cercle correspond tout simplement aux coordonnées du curseur. Quand au rayon, nous prendrons une valeur arbitraire.

Déterminer si un point est dans un cercle revient à calculer la **collision** entre un point et un cercle.

De nombreux tutoriels existent et expliquent les formules de collision. Pour en savoir plus sur les méthodes de collision, jetez un oeil à ce tutoriel : http://www.jeffreythompson.org/collision-detection/point-circle.php

### Approche générale

Nous allons modifier la méthode de rendu afin de vérifier, pour chaque pixel de notre imageData s'il est, ou n'est pas, dans le cercle centré sur la position de notre curseur. Afin de faciliter le calcul, nous allons nous placer dans le repère géométrique de notre canvas.

```js
render: (context, canvas) => {
    const imageData = extractVideoImageData(video, canvas.width, canvas.height);
    
    const coordsRelativeToCanvas = PointerCoordsHelper.getCoordsRelativeToElement(
        canvas,
        pointerCoords.x,
        pointerCoords.y
    );

    const buffer = imageData.data;
    
    // apply to the whole buffer, execept a circle defined by pointer position
    for (let offset = 0; offset < buffer.length; offset += 4) {
        const pixelOffset = (offset / 4); // pixels have 4 channel in ImageData
        const pixelX = pixelOffset % canvas.width;
        const pixelY = pixelOffset / canvas.width;

        // arbitrary radius
        const radius = 50;

        const isInCircle = CollisionHelper.isPointInCircle(
            pixelX, pixelY,
            coordsRelativeToCanvas.x, coordsRelativeToCanvas.y,
            radius
        );

        const grayscale = rgbToGrayscale(buffer, offset);

        if (isInCircle === false) {
            buffer[offset] = grayscale;
            buffer[offset + 1] = grayscale;
            buffer[offset + 2] = grayscale;
            buffer[offset + 3] = 255;
        } else {
            const distance = GeometryHelper.getDistanceBetween2DPoints(
                pixelX, pixelY,
                coordsRelativeToCanvas.x, coordsRelativeToCanvas.y
            );

            const weight = distance / radius;
            // apply a weight in order to let color intensity increase from the outside to the center
            buffer[offset] = weight * grayscale + (1 - weight) * buffer[offset];
            buffer[offset + 1] = weight * grayscale + (1 - weight) * buffer[offset + 1];
            buffer[offset + 2] = weight * grayscale + (1 - weight) * buffer[offset + 2];
            buffer[offset + 3] = 255;
        }
    }


    animation.clear();
    context.putImageData(imageData, 0, 0);
}
```


![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/grayscale-pointer-2.gif?raw=true)

<details>
<summary>⚠️ Remarque sur le calcul de coordonnées ( Avancé) ⚠️</summary>

Le lecteur attentif aura remarqué qu'on calcule la position du curseur relativement à notre canvas (c'est-à-dire telle que l'origine soit le coin supérieur gauche du canvas). 

Ce n'est pas obligatoire, on aurait pu implémenter le même filtre en utilisant directement les coordonnées du pointeur dans la fenêtre, mais les équations auraient étés plus compliquées. 

De plus, l'on est parfois amené à travailler avec des canvas dont la résolution (*pixel théorique*) est différente de sa taille (*pixel physique*). Dans ce cas, vu que notre algorithme itère sur les pixels théoriques du canvas `animation.context.width` ou `animation.canvas.width`. Il faudra modifier les équations pour prendre en compte ce changement de repère supplémentaire…

</details>

## Sa vision est basée sur le mouvement !


![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/movement-1.gif?raw=true)

"Mouais, c'est pas terrible..."

En effet, on peut mieux faire ! Néanmoins, concentrons nous sur le positif, nous avons un début de quelque chose ! Nous parvenons à déceler les contours du perroquet lorsque celui-ci entre en mouvement. Le problème c'est que ses déplacements sont lents, et pas forcément perceptibles d'une frame sur l'autre. De plus, si les mouvements entre deux slices sont minimes, et que nous rafraichissons notre en 60 fps (ce que tentera de faire naturellement notre classe animation), soit une image approximativement toutes les 16ms, et sachant que la persistance rétinienne est de l'ordre de 1/25 de secondes (40 ms), il y a de grande chance pour que le mouvement, bien que visible dans notre canvas, ne nous soit pas perceptible.

### Alternative simple

Plutôt que de se lancer dans un calcul périlleux sur le taux de rafraîchissement optimal, nous allons opter pour une solution bête et méchante : calculer le mouvement en prenant en compte, non pas la dernière frame, mais les N dernières frames.

![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/movement-2.gif?raw=true)


Voici une tentative se basant sur les 5 dernières frames, avec coefficient équivalent à 15. Si les mouvements sont davantage perceptibles, le coût en calcul lui est nettement plus élevé. On passe de 60 fps à un peu plus de 20 fps. Rien de plus normal, on a presque triplé la charge de travail.

Il existe des méthodes permettant d'obtenir un résultat plus propre, et moins gourmant en calcul, mais également moins simple à expliquer. Et vu que ce n'est pas le but premier de cet article, je les passe sous silence. Sachez juste qu'en utilisant des images séparées dans le temps (régulier, par exemple 40 ms) et une simple moyenne au lieu de frame séparées par des itérations de notre boucle de rendu (potentiellement irrégulière est trop courte), on se rapproche d'un comportement naturel ;)

## La vie en bleu 

![Alt Text](https://github.com/qphilippot/tuto/blob/master/apply-filter-on-video/assets/gif/blue-1.gif?raw=true)
