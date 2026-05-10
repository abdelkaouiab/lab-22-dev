# Lab 22 — Programmation Mobile Android avec Java

# Exploitation des capteurs Android

## Objectif général

Ce lab a pour objectif de développer progressivement une application Android capable d’exploiter les capteurs embarqués d’un smartphone Android.

L’application permet :

* d’afficher les capteurs disponibles
* d’obtenir leurs caractéristiques techniques
* d’afficher les mesures en temps réel
* de tracer des graphes dynamiques
* d’utiliser les capteurs de mouvement
* de créer une reconnaissance simple d’activité
* d’implémenter une boussole numérique
* d’ajouter un compteur de pas

---

# Compétences visées

À la fin de ce TP, l’étudiant est capable de :

* Utiliser `SensorManager`
* Manipuler `SensorEventListener`
* Lire les mesures des capteurs Android
* Créer des fragments Android réutilisables
* Dessiner des graphes personnalisés
* Exploiter l’accéléromètre et le gyroscope
* Implémenter une boussole numérique
* Détecter les mouvements de l’utilisateur
* Préserver la batterie avec `unregisterListener()`

---

# Architecture du projet

```text
app/src/main/java/com/example/sensors/
│
├── MainActivity.java
│
├── fragments/
│   ├── SensorsListFragment.java
│   ├── SensorGraphFragment.java
│   ├── MotionSensorFragment.java
│   ├── StepCounterFragment.java
│   ├── CompassFragment.java
│   └── ActivityRecognitionFragment.java
│
├── utils/
│   └── SensorFormatter.java
│
└── views/
    └── LineChartView.java
```

---

# Partie 1 — Affichage des capteurs disponibles

## Création de la classe SensorFormatter

Fichier :

```java
package com.example.sensors.utils;

import android.hardware.Sensor;

public class SensorFormatter {

    public static String format(Sensor sensor) {

        return "Id : " + sensor.getId() + "\n"
                + "Name : " + sensor.getName() + "\n"
                + "Vendor : " + sensor.getVendor() + "\n"
                + "Version : " + sensor.getVersion() + "\n"
                + "Type : " + sensor.getStringType() + "\n"
                + "Int Type : " + sensor.getType() + "\n"
                + "Resolution : " + sensor.getResolution() + "\n"
                + "Power : " + sensor.getPower() + " mA\n"
                + "Maximum Range : " + sensor.getMaximumRange() + "\n"
                + "Min Delay : " + sensor.getMinDelay() + " µs\n";
    }
}
```

## Explication

Cette classe transforme un objet `Sensor` en texte lisible.

Les informations affichées sont :

* nom du capteur
* fabricant
* version
* résolution
* consommation énergétique
* portée maximale
* vitesse minimale d’acquisition

---

# Partie 2 — Liste des capteurs

## SensorsListFragment.java

```java
package com.example.sensors.fragments;

import android.content.Context;
import android.hardware.Sensor;
import android.hardware.SensorManager;
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.LinearLayout;
import android.widget.ScrollView;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.fragment.app.Fragment;

import com.example.sensors.utils.SensorFormatter;

import java.util.List;

public class SensorsListFragment extends Fragment {

    private SensorManager sensorManager;
    private LinearLayout container;

    @Nullable
    @Override
    public View onCreateView(
            @NonNull LayoutInflater inflater,
            @Nullable ViewGroup parent,
            @Nullable Bundle savedInstanceState) {

        ScrollView scrollView = new ScrollView(requireContext());

        container = new LinearLayout(requireContext());
        container.setOrientation(LinearLayout.VERTICAL);

        scrollView.addView(container);

        sensorManager = (SensorManager)
                requireActivity().getSystemService(Context.SENSOR_SERVICE);

        displaySensors();

        return scrollView;
    }

    private void displaySensors() {

        List<Sensor> sensors =
                sensorManager.getSensorList(Sensor.TYPE_ALL);

        for (Sensor sensor : sensors) {

            TextView textView = new TextView(requireContext());

            textView.setText(
                    SensorFormatter.format(sensor));

            container.addView(textView);
        }
    }
}
```

## Explication

Le fragment récupère tous les capteurs disponibles grâce à :

```java
sensorManager.getSensorList(Sensor.TYPE_ALL)
```

Chaque capteur est affiché dynamiquement dans un `TextView`.

---

# Partie 3 — Création du graphe dynamique

## LineChartView.java

```java
package com.example.sensors.views;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Path;
import android.view.View;

import java.util.ArrayList;
import java.util.List;

public class LineChartView extends View {

    private final List<Float> values = new ArrayList<>();

    private final int maxPoints = 80;

    private final Paint axisPaint = new Paint();
    private final Paint linePaint = new Paint();

    public LineChartView(Context context) {
        super(context);

        axisPaint.setColor(Color.LTGRAY);

        linePaint.setColor(Color.BLUE);
        linePaint.setStrokeWidth(5);
        linePaint.setStyle(Paint.Style.STROKE);
    }

    public void addValue(float value) {

        if (values.size() >= maxPoints) {
            values.remove(0);
        }

        values.add(value);

        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {

        super.onDraw(canvas);

        int width = getWidth();
        int height = getHeight();

        canvas.drawLine(40, height - 40,
                width - 20,
                height - 40,
                axisPaint);

        canvas.drawLine(40, 20,
                40,
                height - 40,
                axisPaint);

        if (values.size() < 2) {
            return;
        }

        float min = Float.MAX_VALUE;
        float max = -Float.MAX_VALUE;

        for (float value : values) {
            min = Math.min(min, value);
            max = Math.max(max, value);
        }

        if (max == min) {
            max = min + 1;
        }

        Path path = new Path();

        for (int i = 0; i < values.size(); i++) {

            float x = 40 + i * ((width - 80f)
                    / (maxPoints - 1));

            float normalizedValue =
                    (values.get(i) - min)
                            / (max - min);

            float y = height - 40
                    - normalizedValue * (height - 80);

            if (i == 0) {
                path.moveTo(x, y);
            } else {
                path.lineTo(x, y);
            }
        }

        canvas.drawPath(path, linePaint);
    }
}
```

## Explication

Cette vue personnalisée permet d’afficher un graphe temps réel.

Méthodes importantes :

```java
addValue()
```

Ajoute une nouvelle valeur.

```java
invalidate()
```

Demande à Android de redessiner le graphe.

```java
onDraw()
```

Dessine les axes et la courbe.

---

# Partie 4 — Température, humidité, proximité et champ magnétique

## SensorGraphFragment.java

```java
sensorManager.registerListener(
        this,
        sensor,
        SensorManager.SENSOR_DELAY_NORMAL);
```

## Lecture des valeurs

```java
@Override
public void onSensorChanged(SensorEvent event) {

    float value = extractValue(event.values);

    updateUi(value);
}
```

## Calcul du champ magnétique

```java
return (float) Math.sqrt(
        values[0] * values[0]
                + values[1] * values[1]
                + values[2] * values[2]);
```

## Simulation des capteurs

```java
if (sensorType == Sensor.TYPE_AMBIENT_TEMPERATURE) {

    value = 24f
            + (float) Math.sin(simulationTime / 5f) * 3f;
}
```

## Explication

Le fragment est réutilisable.

Il permet de gérer plusieurs capteurs :

* température
* humidité
* proximité
* champ magnétique

Le graphe se met à jour automatiquement en temps réel.

---

# Partie 5 — Accéléromètre, gravité et gyroscope

## Accéléromètre

```java
Sensor.TYPE_ACCELEROMETER
```

Mesure les accélérations selon :

* axe X
* axe Y
* axe Z

---

## Gravité

```java
Sensor.TYPE_GRAVITY
```

Mesure uniquement la gravité terrestre.

---

## Gyroscope

```java
Sensor.TYPE_GYROSCOPE
```

Mesure la rotation du téléphone en radians/seconde.

---

# Partie 6 — Compteur de pas

## Permission AndroidManifest.xml

```xml
<uses-permission
    android:name="android.permission.ACTIVITY_RECOGNITION" />
```

---

## StepCounterFragment.java

### Déclaration du capteur

```java
stepCounterSensor =
        sensorManager.getDefaultSensor(
                Sensor.TYPE_STEP_COUNTER);
```

---

## Lecture des pas

```java
@Override
public void onSensorChanged(SensorEvent event) {

    float totalStepsSinceBoot = event.values[0];

    if (initialSteps < 0) {
        initialSteps = totalStepsSinceBoot;
    }

    int sessionSteps =
            (int) (totalStepsSinceBoot - initialSteps);

    textView.setText(
            "Pas depuis le dernier redémarrage : "
                    + (int) totalStepsSinceBoot
                    + "\n\nPas de la session : "
                    + sessionSteps);
}
```

---

## Économie d’énergie

```java
@Override
public void onPause() {
    super.onPause();

    sensorManager.unregisterListener(this);
}
```

## Explication

Le compteur de pas Android retourne le nombre total de pas depuis le dernier redémarrage du téléphone.

La variable :

```java
initialSteps
```

permet de calculer uniquement les pas effectués pendant la session.

---

# Partie 7 — Boussole numérique

## CompassFragment.java

### Initialisation des capteurs

```java
accelerometer =
        sensorManager.getDefaultSensor(
                Sensor.TYPE_ACCELEROMETER);

magnetometer =
        sensorManager.getDefaultSensor(
                Sensor.TYPE_MAGNETIC_FIELD);
```

---

## Calcul de l’orientation

```java
boolean success = SensorManager.getRotationMatrix(
        rotationMatrix,
        null,
        gravityValues,
        magneticValues);
```

---

## Conversion en degrés

```java
float azimuthDegrees =
        (float) Math.toDegrees(azimuthRadians);
```

---

## Direction cardinale

```java
if (degree >= 337.5 || degree < 22.5) {
    return "Nord";
}
```

## Explication

La boussole combine :

* l’accéléromètre
* le magnétomètre

pour calculer la direction du téléphone.

---

# Partie 8 — Reconnaissance d’activité

## ActivityRecognitionFragment.java

### Filtre passe-bas

```java
gravity[0] = ALPHA * gravity[0]
        + (1 - ALPHA) * x;
```

---

## Suppression de la gravité

```java
float linearX = x - gravity[0];
float linearY = y - gravity[1];
float linearZ = z - gravity[2];
```

---

## Intensité du mouvement

```java
float movement = (float) Math.sqrt(
        linearX * linearX
                + linearY * linearY
                + linearZ * linearZ);
```

---

## Détection d’activité

```java
if (max > 10f) {
    return "Saut";
}

if (standardDeviation > 1.2f) {
    return "Marche";
}
```

## Explication

Le système détecte plusieurs états :

* marche
* saut
* téléphone stable
* position assise
* position debout

Le système repose sur l’analyse des variations de l’accéléromètre.

---

# Partie 9 — Intégration dans MainActivity

## Méthode d’ouverture des fragments

```java
private void openFragment(Fragment fragment) {

    getSupportFragmentManager()
            .beginTransaction()
            .replace(R.id.fragment_container, fragment)
            .commit();
}
```

---

## Gestion du menu

```java
if (id == R.id.menu_steps) {
    openFragment(new StepCounterFragment());
}

if (id == R.id.menu_compass) {
    openFragment(new CompassFragment());
}

if (id == R.id.menu_activity) {
    openFragment(new ActivityRecognitionFragment());
}
```

## Explication

La méthode `openFragment()` permet de remplacer dynamiquement les écrans dans l’application.

---

# Partie 10 — Tests et validation

## Tests réalisés

### Test des capteurs

Validation de :

* Name
* Vendor
* Version
* Type
* Int Type
* Resolution
* Power
* Maximum Range
* Min Delay

---

### Test température

Observation :

* affichage dynamique
* mise à jour du graphe

---

### Test proximité

Observation :

* valeur proche de 0 lorsque la main approche

---

### Test champ magnétique

Observation :

* variation du champ selon l’orientation

---

### Test accéléromètre

Observation :

* variation des axes x, y, z

---

### Test gyroscope

Observation :

* rotation détectée en rad/s

---

### Test compteur de pas

Observation :

* augmentation du nombre de pas

---

### Test boussole

Observation :

* changement dynamique de direction

---

### Test reconnaissance d’activité

États détectés :

* stable
* marche
* mouvement brusque
* changement d’orientation

---

# Synthèse pédagogique

Ce lab montre comment Android permet d’exploiter les capteurs matériels d’un smartphone.

L’application développée utilise :

* SensorManager
* SensorEventListener
* fragments Android
* vues personnalisées
* graphes dynamiques
* capteurs de mouvement
* reconnaissance simple d’activité

La solution développée reste pédagogique.

Pour une application professionnelle, il serait nécessaire :

* de collecter des données réelles
* d’entraîner un modèle de Machine Learning
* d’évaluer la précision sur plusieurs utilisateurs
* d’optimiser davantage la consommation énergétique

---

# Conclusion

Ce TP a permis de comprendre l’utilisation avancée des capteurs Android avec Java.

Les différentes parties du projet montrent comment :

* récupérer des données matérielles
* traiter les signaux des capteurs
* afficher des graphes temps réel
* reconnaître des mouvements
* créer une interface modulaire avec des fragments

Ce type d’application est utilisé dans :

* les applications sportives
* les objets connectés
* les systèmes de navigation
* les applications médicales
* les systèmes de suivi d’activité

---

# Technologies utilisées

* Java
* Android Studio
* Android SDK
* SensorManager
* SensorEventListener
* Fragments Android
* Canvas API
* Accéléromètre
* Gyroscope
* Magnétomètre
* Step Counter

---

# Auteur

* Nom : [Votre Nom]
* Filière : Génie Informatique / Cybersécurité
* Module : Programmation Mobile Android
* Lab : Lab 22 — Capteurs Android

