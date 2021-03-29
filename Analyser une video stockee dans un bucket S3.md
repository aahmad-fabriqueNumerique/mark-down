# Analyser une vidéo stockée dans un bucket S3

Afin d’analyser une vidéo stockée sur AWS cloude, il faut utiliser les services suivants d’Amazon AWS:
- Rekognition.
- S3

Le service S3 sert à stocker les images des personnes et la vidéo en question.
Rekognition va parcourir tous les fichiers images (jpeg / png) existantes dans un bucket S3 et indexer tous les visages trouvés dans une collection sous form de vecteurs:
```sh
{
    'FaceId': '6d7a3e06-c692-40c0-939e-d709f53c7f2c',
     'BoundingBox':
    {
        'Width': 0.21031899750232697, 
        'Height': 0.5238850116729736, 
        'Left': -0.00038688999484293163, 
        'Top': 0.20262500643730164
    }
}
```
Ensuite, il va parcourir une vidéo et la découper en plusieurs frames selon sa longueur (durée).
Puis, il va détecter les visages dans chaque frame et comparer chaque visage détecté avec les visages paramétrés dans la collection.

Nous avons analysé plusieurs vidéos filmées en différentes conditions:

### **Cas numéro 1**
Vidéo courte de 2 secondes dans laquelle apparaît deux visages filmés de près avec une caméra selfie d’un mobile.
#### *L’analyse*
La vidéo a été découpée en 10 frames.
- Timestamp  0 Image  osm.jpeg Similarity  99.7484359741211
- Timestamp  0 Image alaa.jpeg Similarity  99.9998779296875
- Timestamp  472 Image  osm.jpeg Similarity  99.42264556884766
- Timestamp  472 Image  alaa.jpeg Similarity  99.99990844726562
- Timestamp  977 Image  osm.jpeg Similarity  99.86285400390625
- Timestamp  977 Image  alaa.jpeg Similarity  99.99971771240234
- Timestamp  1483 Image  osm.jpeg Similarity  99.72023010253906
- Timestamp  1483 Image  alaa.jpeg Similarity  99.99967956542969
- Timestamp  1989 Image  osm.jpeg Similarity  99.90322875976562
- Timestamp  1989 Image  alaa.jpeg Similarity  99.99913787841797

**Les deux visages qui apparaissent dans la vidéo ont été reconnus tout au long de la vidéo au même timestamp avec une similarité de plus de 99%.**

### **Cas numéro 2**
Vidéo de 16 secondes dans laquelle apparaît 4 visages filmés de près avec une caméra selfie d’un mobile.

#### *L’analyse*
La vidéo a été découpée en 31 frames.
- Timestamp  0 Image  osm.jpeg Similarity  99.96057891845703
- Timestamp  467 Image  osm.jpeg Similarity  99.97372436523438
- Timestamp  967 Image  osm.jpeg Similarity  99.97142791748047
- ​Timestamp  1468 Image  osm.jpeg Similarity  99.9822998046875
- ​Timestamp  1968 Image  alaa5.jpeg Similarity  99.99977111816406
- ​Timestamp  2469 Image  alaa5.jpeg Similarity  99.99983978271484
- ​Timestamp  2970 Image  alaa5.jpeg Similarity  99.99836730957031
- ​Timestamp  3470 Image  alaa5.jpeg Similarity  99.99563598632812
- Timestamp  3971 Image  alaa5.jpeg Similarity  99.99729919433594
- Timestamp  4471 Image  alaa5.jpeg Similarity  99.98868560791016
- Timestamp  4972 Image  alaa5.jpeg Similarity  99.99677276611328
- Timestamp  5473 Image  alaa5.jpeg Similarity  90.9659423828125
- Timestamp  5973 Image  alaa5.jpeg Similarity  99.99824523925781
- Timestamp  6474 Image  alaa5.jpeg Similarity  92.88441467285156
- Timestamp  6974 Image  alaa5.jpeg Similarity  99.92341613769531
- Timestamp  7475 FaceMatches  undefined
- Timestamp  7975 FaceMatches  undefined
- Timestamp  8476 FaceMatches  undefined
- Timestamp  8977 FaceMatches  undefined
- Timestamp  9477 FaceMatches  undefined
- Timestamp  9978 FaceMatches  undefined
- Timestamp  10478 FaceMatches  undefined
- Timestamp  12481 FaceMatches  undefined
- Timestamp  12981 FaceMatches  undefined
- Timestamp  13482 FaceMatches  undefined
- Timestamp  13982 FaceMatches  undefined
- Timestamp  14483 FaceMatches  undefined
- Timestamp  14984 FaceMatches  undefined
- Timestamp  15484 FaceMatches  undefined
- Timestamp  16485 Image  osm.jpeg Similarity  98.17233276367188
- Timestamp  16986 Image  osm.jpeg Similarity  97.14662170410156

**Trois visages ont été reconnus avec une similarité de plus 99% sauf dans le timestamp (5473) où la caméra s’éloignait de la personne (90%) . Le quatrième visage n’a pas été reconnu car il n'existe pas dans la collection d’où le résultat:  “FaceMatches  undefined”.**

### **Cas numéro 3**
Vidéo de 19 secondes (video 5) dans laquelle apparaissent 2 enfants filmés de près avec la caméra arrière d’un mobile.

#### *L’analyse*
La vidéo a été découpée en 44 frames.
….
- Timestamp  11980 FaceMatches  undefined
- Timestamp  12481 Image  osm.jpeg Similarity  85.18231201171875
- Timestamp  12481 Image  celina.jpeg Similarity  99.17035675048828
….
- Timestamp  14984 Image  eliana.jpeg Similarity  98.78900146484375
….

**Dans cette vidéo apparaissent deux personnes (visages) mais Rekognition a détecté un troisième visage sur le reflet du vitre avec une similarité de 85%.**

### **Cas numéro 4**
Vidéo de 14 secondes (video 7) dans laquelle apparaissent 4 personnes filmées dans l’ombre en s’approchant de loin avec la caméra arrière d’un mobile .

#### *L’analyse*
La vidéo a été découpée en 38 frames.
...
- Timestamp  3468 FaceMatches  undefined
- Timestamp  3968 Image  mathieuP Similarity  95.75979614257812
- Timestamp  3968 FaceMatches  undefined
 -Timestamp  4468 Image  mathieuP Similarity  81.27916717529297
- Timestamp  4468 Image  mathieuH Similarity  97.37427520751953
- Timestamp  4468 Image  jmc Similarity  97.7589111328125
- Timestamp  4968 Image  mathieuP Similarity  90.3725814819336

 **Les quatre personnes ont été détectées avec une similarité en dessus de 90%.**

### **Cas numéro 5**
Vidéo de 29 secondes (video 7) dans laquelle apparaissent 5 personnes filmées sous le soleil et la caméra tournait autour d’eux .
#### *L’analyse*
La vidéo a été découpée en 61 frames.
….
- Timestamp  7991 Image  alaa Similarity  99.99310302734375
- Timestamp  7991 Image  jmc Similarity  99.9293212890625
- Timestamp  7991 FaceMatches  undefined
- Timestamp  8490 Image  osm.tahar Similarity  99.27716827392578
- Timestamp  8490 Image  alaa Similarity  99.99060821533203
- Timestamp  8490 Image  jmc Similarity  99.9725112915039
- Timestamp  8490 FaceMatches  undefined
….

**Tous les visages ont été détectés avec une similarité de plus de 99%.**

#### **Nous constatons qu'il suffit d’avoir un certain nombre de photos (max 5 photos par personnes) pour que Rekognition puisse reconnaître cette personne avec une confiance de plus de 99%.**



