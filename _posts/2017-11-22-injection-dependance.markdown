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

En effet, dans ce cas, modifier le type déclaré des champs, paramètres, variables, etc. est simple et réduit le couplage: 
ces propriétés ne dépendront que de l'interface `List` et non de ses différentes implémentation.
Cependant, lors de l'initialisation d'un champ ou d'une variable, vous ne pouvez faire autrement qu'utiliser la classe concrète:

```java
public  MyClass() {
  tags = new ArrayList<String>();
}
```

Cette unique utilisation de `ArrayList` empêche la classe `MyClass` d'être complètement indépendante des classes concrètes: elle continue de
dépendre de l'interface `List` et de son implémentation.

Une solution simple à ce problème, c'est d'instancier la liste d'éléments à l'extérieur de la classe et de la passer comme paramètre du constructeur:

```java
public class MyClass {
  private List<String> tags;
  public MyClass(List l) {
    this.tags = l;
  }
}
```
Si cette solution résout partiellement le problème et peut améliorer la testabilité de la classe: les cas de test pourront la configurer de 
l'extérieur et ensuite contrôler et observer son comportement, grâce aux doublures de test.

Mais la solution rend l'utilisation de la classe plus complexe:
tout client souhaitant l'utiliser devra d'abord instancier une liste et la passer comme argument.
Et si plusieurs champs doivent être instanciés à l'extérieur de la classe, le code deviendra trop complexe.

Une autre solution similaire et tout aussi partielle, consiste à utiliser une méthode pour initialiser la liste d'éléments:

```java
public class MyClass {
  private List<String> tags;
  public void setTags(List l) {
    this.tags = l;
  }
}
```

Cette solution non seulement présente les mêmes inconvénients  que la précédente, comme aussi en introduit un nouveau:
le client doit être au courant que cette méthode doit être appelée avant l'utilisation effective d'une instance de cette classe.

## Le pattern «Factory» à la rescousse

Une solution intéressante pour simplifier l'initialisation de la classe `MyClass` est apportée par les patrons de conception.
En effet, le patron _Factory_ permet de simplifier la création d'objets dont la configuration est complexe. 
L'idée de ce pattern est d'utiliser une autre classe ayant pour seul objectif la création d'objects dont elle est responsable 
pour les fournir prêt-à-servir; la classe concrète utilisée n'étant pas connue par l'appelant.

Une implémentation possible de la solution est listée ci-dessous:



```java
public class MyClassFactory {
    public MyClass createMyClass() {
      return new MyClass(new ArrayList());
    }
}
```

Ici, la fabrique se charge de configurer la classe `MyClass` lors de sa création. 
`MyClass` devient enfin complètement indépendante de l'implémentation de la classe `List`.

Une autre solution pour simplifier l'utilisation de MyClass consiste à utiliser l'injection de dépendance,
qui nous permettra de nous passer des fabriques, tout en gardant les avantages.


## Injection de dépendance

L'injection de dépendance consiste à faire appel à un mécanisme externe à la classe, qui va se charger d'initialiser ses attributs,
ou en d'autres termes, va se charger d'injecter automatiquement des dépendances vers des classes concrètes. 
Il existe plusieurs mécanismes d'injection de dépendance. Un de plus connus et aussi le plus utilisé est [Spring](https://spring.io),
mais d'autres mécanismes plus légers et récents sont aussi disponibles: [Google Guice](https://github.com/google/guice),
[Dagger](http://square.github.io/dagger/), [PicoContainer](http://picocontainer.com/) ou [Weld](http://weld.cdi-spec.org).
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
public class MyConfiguration {

    @Bean
    public MyClass myClass() {
        return new MyClass();
    }

    @Bean
    public List<String> produceList() {
        return new LinkedList<String>();
    }
}
```

Dans cet exemple, le type de retour de la méthode `produceList()` fait de celle-ci la seule candidate pour initialiser la variable `tags`.

Notez que cette configuration n'est pas spécifique à la classe `MyClass`, elle peut servir à d'autres classes. 

### Utilisation

L'utilisation de la classe `MyClass` doit passer par un _Contexte_, qui utilisera la configuration pour créer des instances de `MyClass`

```java
public static void main(String[] args) {

    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyConfiguration.class);
    MyClass mine = (MyClass) ctx.getBean(MyClass.class);
}
```

Lorsque le contexte initialise une instance de `MyClass`, grâce à la méthode `getBean()`, il initialisera automatiquement l'attribut `tags` grâce à la
méthode `produceList()` de la classe `MyConfiguration`.
Ici, le nom de la méthode n'est pas important, ce qui compte c'est le type de retour. Ainsi, pour initialiser un attribut de la classe `List`,
Spring cherchera les méthodes de création (`@Bean`) dont le type de retour est compatible avec `List`.


On peut utiliser cette configuration également pour initialiser l'attribut avec une autre implémentation de `List` et même pour ajouter quelques valeurs à cette liste.

### Code source

Le code source de cet exemple est disponible sur le [GitHub](https://github.com/sunye/blog-examples/tree/master/dependency-injection).

## Conclusion

Grâce à l'injection de dépendance, il est possible de séparer l'initialisation des attributs de leur utilisation.
Dans notre exemple, les conséquences sont les suivantes:

1. La classe `MyClass` ne connaît pas la classe concrète que l'attribut `tags` utilise: elle ne dépend que de l'interface `List`.
1. Les classes qui utilisent `MyClass` (les clientes) n'ont pas besoin de la configurer pour l'utiliser.
1. La liaison entre l'attribut de type `List` et la classe concrète `LinkedList` est fait par une classe tiers de configuration.
1. Lorsqu'on utilise Spring pour l'injection de dépendance, il est possible de remplacer la classe de configuration par un fichier XML. 

