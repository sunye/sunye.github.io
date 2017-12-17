---
layout: post
title: "Analyse de la mutation avec PIT"
categories: Java PIT
date:   2017-12-15 11:00:00 +0100
---

## Introduction 

Vous vous rappelez de votre cours de physique où votre professeur a demandé à toute la classe la température d'ébullition de l'eau et que vous, 
fier de votre culture vous avez répondu:

> -- 100!

Et quand, sûr de vous,  vous attendiez les lauriers de la victoire et le regard admiratif de vos camarades, votre professeur a crié:

> -- 100 quoi? Patates? Les unités de mesure, ventrebleu!

Tous vos camarades ont rit et vous avez rougi. 
C'est à ce moment que vous avez commencé à éviter vos camarades, les gens en générale, et là, deux solutions s'offraient à vous: aller voir un psy ou devenir informaticien pour développer un programme qui ferait que plus personne n'oublierait les unités! Et ainsi, aucune sonde spatiale ne tomberait par votre faute[^1]. 
Et vous vous êtes mis à développer votre propre classe `Mètre` que vous comptez utiliser toute votre vie. 

## La classe «Mètre»

Vous avez commencé à coder votre classe et avez ajouté une première méthode, `add()`, qui permet l'addition de 2 distances.
Bien que son comportement soit simple, cette méthode assure, par exemple, que personne ne pourra faire une addition entre une distance
mesurée en mètres et une autre mesurée en yards:

```java
class Meter {
    private int length;
	public Meter add(Meter other) {
		return new Meter(this.length + other.length);
	}
}
```

Comme vous êtes soucieux de la qualité de votre code, vous avez écrit un test unitaire pour vérifier votre méthode,
en utilisant le framework de tests [JUnit](http://junit.org/), combiné avec des assertions [AssertJ](http://joel-costigliola.github.io/assertj/):

```java
@Test
void testAdd() {
    Meter m2 = new Meter(2);
    assertThat(m2.add(m2)).isEqualTo(new Meter(4));
}
```

Après avoir exécuté votre test avec succès, vous vous êtes posé une question:

> -- Est-ce que mon test est suffisant ?

Votre premier réflexe a été de vous retourner vers la couverture de code et à des outils comme [Cobertura](http://cobertura.github.io/cobertura/) 
ou [JaCoCo](http://www.jacoco.org/jacoco/).
Et le résultat était parfait: votre test couvre 100% des instruction (critères «tous-les-noeuds» et «tous-les-arcs»).
C'est la réussite totale!

Mais votre esprit critique n'était pas satisfait de cette réponse. Car vous avez appris que couvrir 100% des instructions est une bonne chose,
mais cela ne veut pas dire que votre test détecte toutes les erreurs pour autant. 
Et que 1 seul test, avec 1 seule donnée de test vous paraît bien insuffisant. 

> -- Comment puis-je évaluer _vraiment_ la qualité de mes tests? vous êtes-vous demandé une après-midi pendant le goûter. 

Et en regardant le «Blade Runner» Rick Deckard faire passer le test d'empathie [Voight-Kampff](https://fr.wikipedia.org/wiki/Voight-Kampff) pour détecter des _réplicants_, vous vous êtes demandé: et pourquoi pas l'analyse de mutation? 

## Analyse de mutation

L'analyse de mutation est une technique de qualification de cas de tests, qui se base sur l'injection de fautes.
Le principe est simple: on injecte des fautes dans le logiciel sous test et si les tests sont capables de détecter ces fautes, alors leur qualité est bonne.
Et s'ils ne détectent pas ces erreurs, alors ils sont de mauvaise qualité.

Pour simplifier l'évaluation des cas de test et en même temps rendre difficile la détection d'erreurs, on injecte une faute à la fois. 
On appelle «mutant» une copie du logiciel sous test à laquelle on a injecté une et une seule faute.

Le choix des fautes à injecter est crucial, car elle ne doivent pas être détectées facilement, mais en même temps, doivent être des vraies fautes.
N'importe quel test peut détecter une erreur d'exécution, comme une division par zéro ou un index hors-plage, par exemple.


On appelle un «opérateur de mutation» une opération qui injecte une faute dans un programme.
Il peut être simple, comme le remplacement d'un opérateur arithmétique `(+,-,*,/,..)` par un autre. 
Il y a aussi des opérateurs de mutation beaucoup plus complexes, comme l'ajout d'un champ (attribut) qui cacherait un champ hérité d'une super-classe. 

## Les mutants

Si l'on applique des opérateurs de mutation arithmétiques à la méthode `add()`, on obtient (au moins) quatre mutants.
Tout d'abord, le remplacement de l'addition par une soustraction:

```java
// Cyclope
public Meter add(Meter other) {
    return new Meter(this.length - other.length);
}
```
Ensuite, remplacement par une multiplication:
```java
// Iceberg
public Meter add(Meter other) {
    return new Meter(this.length * other.length);
}
```
Et par une division:
```java
// Mimic
public Meter add(Meter other) {
    return new Meter(this.length / other.length);
}
```
Et finalement, le remplacement de l'addition par l'opération modulo, le reste d'une division euclidienne:
```java
// Polaris
public Meter add(Meter other) {
    return new Meter(this.length % other.length);
}
```

Notez que si jamais la méthode `add()` contenait une deuxième opération d'addition, on obtiendrait 4 autres mutants, soit 8 au total.

## À la chasse des mutants

Grâce à ces 4 mutants, on peut qualifier notre test `testAdd()`.
Plus précisément, on va tester chacun des mutants.
Si le verdict de l'exécution est un «échec», le test a bien détecté le mutant.
À l'opposé, si le verdict du test est un «succès», le test n'a pas été capable de détecter le mutant.

Lorsqu'on teste les quatre mutants, on obtient le résultat suivant:

Mutant | Résultat évalué | Résultat attendu | Verdict | État du mutant
|---|:---:|:---:|:---:|:---:|
*Cyclope* | 0m | 4m | Échec  | Mort
*Iceberg* | 4m | 4m | Succès | Vivant
*Mimic*   | 1m | 4m | Échec  | Mort
*Polaris* | 0m | 4m | Échec  | Mort

Le cas de test a été capable de détecter 3 mutants sur 4. 
Il a été capable de tuer _Cyclope_, _Mimic_ et _Polaris_, mais _Iceberg_ reste vivant.
On parle alors d'un «score de mutation» de 75%. 

En d'autres termes, votre cas de test est bon, mais pas aussi bon que la couverture de code laissait croire.
Vous aurez besoin d'un deuxième cas de test pour atteindre un score de mutation de 100%. 
Par exemple, le test suivant vous permettra d'atteindre un score de mutation de 100%:

```java
@Test
void testAddBis() {
    Meter m2 = new Meter(2);
    Meter m4 = new Meter(4);
    assertThat(m2.add(m4)).isEqualTo(new Meter(6));
}
```

## Limites de l'analyse de mutation

Si les résultats de l'analyse de mutation sont plus pertinents que ceux de la couverture de code, son coût est bien plus important. 
Le nombre important de mutants créés pour le logiciel sous test, combinés au temps d'exécution du jeu de tests peuvent rendre l'analyse de mutation impraticable.

De plus, l'analyse de mutation, tout comme la couverture de code, ne peut pas dire si un comportement absent (non mis en œuvre) est correctement testé.
Par exemple, votre méthode `add()` contient une erreur importante, elle ne gère pas le débordement d'entiers.
Si le résultat de l'addition est supérieur au plus grand d'entier (`Integer.MAX_VALUE`), Java rendra un nombre négatif. 

Les techniques de [test fonctionnel](https://sunye.github.io/testing-courseware/#/5) et plus précisément le test aux limites, peuvent révéler cette erreur.

## PIT

[PIT](http://pitest.org) est un outil d'analyse de mutation capable de qualifier des suites de test écrites en JUnit. 
Il a été développé par [Henry Coles](https://github.com/hcoles) et est distribué sous licence Apache 2.0.

Pour réduire le temps d'exécution et limiter le nombre de mutants générés, PIT combine l'analyse de mutation avec la couverture de code.
Les opérateurs de mutation ne sont appliqués qu'au code couvert par les tests unitaires: PIT ne modifie pas le code qui n'est pas exécuté par les tests.
Aussi, PIT n'exécute que les cas de test qui sensibilisent le code modifié.

Par exemple, votre test unitaire `testAdd()` n'exécute que le code de la méthode `add()`.
Si PIT crée une centaine de mutants, dont seulement 4 concernent la méthode `add()`, le test `testAdd()` ne sera exécuté que 4 fois.

PIT s'intègre parfaitement à l'environnement de construction Maven-Gradle-JUnit.
Pour l'utiliser, ajoutez la dépendance suivante à votre fichier `pom.xml`:

```xml
<dependency>
    <groupId>org.pitest</groupId>
    <artifactId>pitest</artifactId>
    <version>1.2.5</version>
    <type>pom</type>
</dependency>
```

Ensuite, ajoutez la configuration du plugin `pitest-maven` à la phase «build» de Maven, 
en spécifiant les paquetages Java qui seront modifiés par PIT:

```xml
<plugin>
 <groupId>org.pitest</groupId>
 <artifactId>pitest-maven</artifactId>
 <configuration>
     <targetClasses>
         <param>fr.unantes.info.units.*</param>
     </targetClasses>
     <targetTests>
         <param>fr.unantes.info.units.*</param>
     </targetTests>
 </configuration>
 <executions>
     <execution>
         <goals>
             <goal>mutationCoverage</goal>
         </goals>
         <phase>pre-site</phase>
     </execution>
 </executions>
</plugin>
```

Lors que vous utilisez Maven pour créer les rapports de votre projet avec `mvn site`, PIT crée un rapport contenant les résultats de l'analyse de mutation.

## Notes
[^1]: Mars Climate Orbiter (1998) était une sonde qui s'est écrasée sur Mars en raison d'une [erreur logicielle liée aux unités de mesure](http://www.nirgal.net/mco_end.html).

