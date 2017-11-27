---
layout: post
title:  "Injection de dépendance"
date:   2017-11-21 13:52:25 +0100
categories: Java Spring
---

## Introduction

Vous vous êtes déjà fait taper sur les doigts par votre instructeur de Java préféré lorsque vous avez déclaré une liste d'éléments de la façon suivante:

```java
public class MyClass {
  private ArrayList<String> tags;
}
```
Vous n'avez peut-être pas compris, mais il avait raison. 
Votre code ne doit pas dépendre d'une classe concrète lorsqu'il n'a besoin que des méthodes déclarés par une interface, comme c'est le cas ici.
Vous avez donc modifié votre code, pour essayer de ne dépendre que de l'interface `List`:

```java
public class MyClass {
  private List<String> tags;
}
```

En effet, dans ce cas, modifier le type déclaré des champs, paramètres, variables, etc. est simple et réduit le couplage.
Cependant, lors de l'initialisation d'un champ ou d'une variable, vous ne pouvez faire autrement qu'utiliser la classe concrète:

```java
public  MyClass() {
  tags = new ArrayList<String>();
}
```

Une solution simple à ce problème, c'est d'instancier la liste d'éléments à l'extérieur de la classe et de la passer comme paramètre du constructeur:

```java
public class MyClass {
  private List<String> tags;
  public MyClass(List l) {
    this.tags = l;
  }
}
```
Si cette solution résout partiellement le problème et peut améliorer la testabilité de la classe, elle rend son utilisation plus complexe:
tout client souhaitant l'utiliser devra d'abord instancier une liste et la passer comme argument.
Et si plusieurs champs doivent être instanciés à l'extérieur de la classe, le code deviendra trop complexe.

Une autre solution similaire et tout aussi partielle, consiste à utiliser une méthode pour l'initialiser la liste d'éléments:

```java
public class MyClass {
  private List<String> tags;
  public void setTags(List l) {
    this.tags = l;
  }
}
```

Cette solution non seulement présente les mêmes inconvenants que la précédente, comme aussi en introduit un nouveau:
le client doit être au courant que cette méthode doit être appelée avant l'utilisation effective d'une instance de cette classe.

## Injection de dépendance

L'injection de dépendance consiste à faire appel à un mécanisme externe à la classe, qui va se charger d'initialiser ses attributs,
ou en d'autres termes, va se charger d'injecter automatiquement des dépendances vers des classes concrètes. 
Il existe plusieurs mécanismes d'injection de dépendance. Un de plus connus et aussi le plus utilisé est [Spring](https://spring.io),
mais d'autres mécanismes plus légers et récents sont aussi disponibles: [Google Guice](https://github.com/google/guice),
[Dagger](http://square.github.io/dagger/) ou [PicoContainer](http://picocontainer.com/).
Dans nos exemples, nous utiliserons Spring.


### Spécification du point d'injection

Dans notre exemple, le rôle de l'injection de dépendance sera d'initialiser automatiquement l'attribut `tags`.
Cependant, il n'y a pas de magie: on devra spécifier quels attributs doivent être initialisés et comment.
Pour ce faire, on utilisera deux spécificités du langage Java: les annotations et la réflexion.
Tout d'abord, nous allons utiliser une des annotations définies par la [JSR-330](http://javax-inject.github.io/javax-inject/),
`@Inject` pour spécifier que l'attribut `tags` doit être initialisé automatiquement:


```java
import javax.inject.Inject;

public class MyClass {
  @Inject
  private List<String> tags;
}
```

### Configuration

Ensuite, la réflexion sera utilisée par une _Configuration_ (dans le jargon Spring) 
ou un _Module_  (dans le jargon Guice)
pour trouver les attributs qui seront initialisés automatiquement, ainsi que leur types, pour ensuite les initialiser.
Dans le cas de Spring, la configuration devra spécifier une méthode compatible avec le type de l'attribut `tags`, `List<String>`.
Voici un exemple de configuration Spring:

```java
@Configuration
public class MyClassConfiguration {

    @Bean
    public MyClass myClass() {
        return new MyClass();
    }

    @Bean
    public List<String> getTags() {
        return new LinkedList<String>();
    }
}
```

Dans cet exemple, le type de retour de la méthode `getTags()` fait de celle-ci la seule candidate pour initialiser la variable `tags`.

Notez que cette configuration n'est pas spécifique à la classe `MyClass`, elle peut servir à d'autres classes. 

### Utilisation

L'utilisation de la classe `MyClass` doit passer par un _Contexte_, qui utilisera la configuration pour créer des instances de `MyClass`

```java
public static void main(String[] args) {

    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyClassConfiguration.class);
    MyClass mine = (MyClass) ctx.getBean(MyClass.class);
}
```

Lorsque le contexte initialise une instance de `MyClass`, grâce à la méthode `getBean()`, il initialisera automatiquement l'attribut `tags` grâce à la
méthode `getTags()` de la classe `MyClassConfiguration`.
On peut utiliser cette configuration également pour initialiser l'attribut avec une autre implémentation de `List` et même pour ajouter quelques valeurs à cette liste.

### Code source

Le code source de cet exemple est disponible sur le [GitLab](https://gitlab.univ-nantes.fr/sunye-g/exemples-spring/tree/master/dependency-injection) de l'Université de Nantes.

## Conclusion

Grâce à l'injection de dépendance, il est possible de séparer l'initialisation des attributs de leur utilisation.
Dans notre exemple, les conséquences sont les suivantes:

1. La classe `MyClass` ne connaît pas la classe concrète que l'attribut `tags` utilise: elle ne dépend que de l'interface `List`.
1. Les classes qui utilisent `MyClass` (les clientes) n'ont pas besoin de la configurer pour l'utiliser.
1. La liaison entre l'attribut de type `List` et la classe concrète `LinkedList` est fait par une classe tiers de configuration.
1. Lorsqu'on utilise Spring pour l'injection de dépendance, il est possible de remplacer la classe de configuration par un fichier XML. 

