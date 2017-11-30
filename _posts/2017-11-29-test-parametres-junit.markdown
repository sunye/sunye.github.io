---
layout: post
title:  "Tests paramétrés en JUnit 5"
date:   2017-11-29 11:00:00 +0100
categories: Java JUnit
---

## Introduction

Ernest Hemingway suggère que pour devenir un être humain accompli, une personne doit réaliser 4 tâches:
1) Planter un arbre, 2) Combattre un taureau, 3) Écrire une nouvelle, 4) Élever un enfant
et 5) Développer sa propre classe Date[^1] en Java. 

Si vous voulez accomplir la dernière, vous aurez sans doute besoin de tester unitairement les méthodes de cette classe.
Et rien de plus adapté pour écrire des tests unitaires en Java que [JUnit](http://junit.org/) dans sa version la plus récente.

## La classe Date

La classe `Date` que nous allons tester a un constructeur et deux méthodes: `isLeapYear(int)` et `int dayOfWeek()`.
Cette classe n'est valable que pour les dates postérieures à l'année 1582 et en France.

Pourquoi seulement à partir de cette date? Parce que l'année 1582 est un peu spéciale: elle a durée 341 jours en raison
de l'adoption du calendrier grégorien par Henri III le 9 décembre (jour qui précéda le 20 décembre 1582).
La réforme a été mise en place en France quelques mois après l'Espagne, le Portugal et la Pologne, mais quelques années
avant le Royaume Uni, qui a attendu jusqu'à 1752 pour le faire, et la Russie, qui ne l'a adoptée qu'en 1918.
Pendant plusieurs années, les pays européens n'avaient pas le même calendrier.
Cette confusion a servi de contexte au roman «Le pendule de Foucault» de Umberto Eco, où des sociétés secrètes se sont perdues après un rendez-vous manqué pendant ces années.

Mais revenons à la méthode `isLeapYear(int)`. Une première méthode de test pourrait être écrite comme suit:

```java
    @Test
    void testTwoThousandIsALeapYear() {
        assertThat(Date.isLeapYear(2000)).isTrue();
    }
```

Bien que capable de détecter 1 erreur de codage (si l'année 2000 n'est pas considérée bissextile), cette méthode de test est insuffisante pour détecter d'autres erreurs.
À défaut de pouvoir la tester exhaustivement, on peut utiliser [l'analyse partitionnelle](https://sunye.github.io/testing-courseware/#/5/13) pour établir des classes d'équivalence et proposer des données de test efficaces.
Par exemple:

  - Années bissextiles: {1900, 1940, 1996, 2000, 2004}
  - Années non bissextiles: {1958,1962,1970,1994,2002}

Certaines années sont parfois considérés comme bissextiles, alors que ce n'est pas le cas. 
Par exemple, les années 1700, 1800 et 1900 ne sont pas bissextiles (parce que divisibles par 100), alors que 1600 et 2000 le sont (car divisibles par 400).
Cela nous permet de trouver 2 autres classes  d'équivalence:

  - Années bissextiles inhabituelles: {1600, 2000}
  - Années non bissextiles inhabituelles: {1700,1800,1900}

Pour utiliser ces différentes données de test, nous allons améliorer notre première méthode de test et la rendre paramétrable.

## Tests paramétrés

En Junit 5, les méthodes de test peuvent avoir des paramètres.
Dans ce cas, les méthodes doivent être ornées par l'annotation `@ParameterizedTest`, comme suit:

```java
    @DisplayName("Valid leap years")
    @ParameterizedTest
    @ValueSource(ints = {1904, 1940, 1996, 2000, 2004})
    void testLeapYear(int year) {
        assertThat(Date.isLeapYear(year)).isTrue();
    }
```
Cette méthode de test est aussi ornée par l'annotation `@ValueSource` qui dans ce cas, permet de spécifier 5 entiers qui serviront de données de test.
JUnit exécutera cette méthode 5 fois, une pour chaque donnée de test.

Notez que l'annotation `@DisplayName` permet de spécifier un nom lisible à la méthode de test.
Notez aussi l'utilisation de [AssertJ](http://joel-costigliola.github.io/assertj/index.html), une des bibliothèques compatibles avec JUnit qui simplifier l'écriture d'assertions.

Nous utilisons ces mêmes annotations pour écrire deux autres méthodes de test pour les années inhabituelles:

```java
    @DisplayName("Unusual valid leap years")
    @ParameterizedTest(name = "year: {0}")
    @ValueSource(ints = {1600,2000})
    void testUnusualLeapYear(int year) {
        assertThat(Date.isLeapYear(year)).isTrue();
    }

    @DisplayName("Unusual invalid leap years")
    @ParameterizedTest(name = "year: {0}")
    @ValueSource(ints = {1700,1800,1900})
    void testUnusualNotLeapYear(int year) {
        assertThat(Date.isLeapYear(year)).isFalse();
    }
```

## Méthodes fournisseuses de données

Une alternative au passage de valeurs par l'annotation `@ValueSource` est d'appeler une méthode qui retourne un `Stream` de données de test. 
La méthode doit être définie comme `static` et son type de retour doit être compatible avec celui du paramètre de la méthode de test.
La liaison entre la méthode de test est la méthode fournisseuse de données se fait par l'annotation `@MethodSource`, qui spécifie le nom de la méthode appelée:

```java
    @DisplayName("Invalid leap years")
    @ParameterizedTest
    @MethodSource("fiveTimesWorldChampion")
    void testNotLeapYear(int year) {
        assertThat(Date.isLeapYear(year)).isFalse();
    }

    static IntStream fiveTimesWorldChampion() {
        return IntStream.of(1958,1962,1970,1994,2002);
    }
```

Pour tester le constructeur de la classe date, `Date(int,int,int)` nous à nouveau appliquer l'analyse partitionnelle pour établir deux classes d'équivalence:

- Dates valides: {10/10/2000, 23/2/1999}
- Dates invalides: {10/10/1000, 33/2/1999, 1/13/1999, 0/2/1999, 10/0/1999}

Comme le constructeur de la classe sous test prend 3 paramètres, nous allons définir une méthode de test avec 3 paramètres également:

```java
    @DisplayName("Constructor: valid dates")
    @ParameterizedTest
    @MethodSource("validDates")
    void testConstructorValidDates(int d, int m, int y) {
        new Date(d,m,y);
        assertThat(date.year()).isEqualTo(y);
        assertThat(date.month()).isEqualTo(m);
        assertThat(date.day()).isEqualTo(d);
    }
```

Quand la méthode de test déclare plusieurs paramètres, le données fournies doivent être du type `Argument`.
Argument est un n-uplet, où le type de chaque composant correspond au type de chaque paramètre de la méthode de test.
Dans notre exemple, les trois composants sont de type entier:

```java
    static Stream<Arguments> validDates() {
        return Stream.of(
                Arguments.of(10, 10, 2000),
                Arguments.of(23, 2, 1999)
        );
    }
```

Une troisième et dernière façon de passer des données à une méthode de test consiste à utiliser l'annotation `@CsvSource`, qui permet de spécifier un ensemble de chaînes de caractères, où chaque chaine contient des arguments au format CSV (séparés par des virgules):

```java
    @DisplayName("Constructor: invalid dates")
    @ParameterizedTest
    @CsvSource({"10,10,1000","33,2,1999","1,13,1999","0,2,1999","10,0,1999"})
    void testConstructotInvalidDates(int d, int m, int y) {
        assertThrows(IllegalArgumentException.class, () -> new Date(d,m,y));
    }
```

Notez que cette méthode de test utilise l'assertion `assertThrows()`, introduite dans la version 5 de JUnit et qui fait appel aux expressions lambda de Java 8.
Son comportement est simple: elle attend qu'une exception du type passé en premier paramètre soit levée lors de l'évaluation de l'expression passée en deuxième paramètre. 

## Code source

Le code source de cet exemple est disponible sur le [GitLab](https://gitlab.univ-nantes.fr/sunye-g/exemples-blog/tree/master/parameterized-tests) de l'Université de Nantes.

## Conclusion

JUnit permet la séparation entre les méthodes de test et les données de tests:
- Les méthodes de test peuvent avoir des paramètres;
- Les données de test sont décrites soit à l'intérieur d'une annotation, soit par une méthode externe, appelée _fournisseuse de données_.
- Les méthodes de test sont appelées autant de fois qu'il y a de données de test.

## Notes

[^1]: Cette dernière suscite de vives controverses.