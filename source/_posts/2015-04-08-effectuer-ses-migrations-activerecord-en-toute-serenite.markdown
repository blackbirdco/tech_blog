---
layout: post
title: "Effectuer ses migrations ActiveRecord en toute sérénité"
date: 2015-04-08 17:41:34 +0100
comments: true
categories: activerecord rails
---

Les techniques de développement agiles amène régulièrement à remanier et
remodeler nos bases de données relationnelles, d'autant plus si les outils
à nos dispositions nous permettent d'effectuer des migrations facilement.

Il est ainsi plus aisé d'introduire des migrations de
données peu robustes au sein de l'application, ce qui peut amener
à des erreurs lors des déploiements, ou pire, à une perte de l'intégrité
de la base de données.

Cette article regroupe **quelques conseils et astuces pour
effectuer vos migrations en toute sérénité**.

<!-- more -->

## Dans le doute : effectuez une sauvegarde

Avant toute chose, n'hésitez surtout pas à effectuer
des backups de votre BDD. Il existe un excellent
outil dans le monde ruby pour ça : [Backup](https://github.com/meskyanichi/backup).

## Les migrations ruby sont des classes

Comme toutes les classes de votre application, *vos
migrations se doivent d'être le plus lisible
possible* :

* Séparez vos opérations dans des méthodes atomiques ;
* Utilisez des noms de méthodes et de variables clairs et conçis ;
* Commentez vos opérations complexes (requêtes SQL, itérations sur les tuples
  d'une relation, etc ...).

## Rendez vos migrations "rollbackable"

Il ne faut pas hésiter, pour chaque migration introduite dans votre application, à
[vérifier que celle-ci est "rollbackable" à l'aide de la tâche rake `db:rollback`](http://robots.thoughtbot.com/workflows-for-writing-migrations-with-rollbacks-in-mind).

Cependant, certaines migrations ne peuvent tout simplement pas être annulées
après leur exécution (typiquement des retraits de colonnes dans une relation).

Dans ces cas là, il est toujours possible d'annuler la migration courante (dans la
méthode `up` de `ActiveRecord::Migration`) à l'aide de l'exception
`ActiveRecord::Rollback`.

A l'aide de cette astuce, il est possible de vérifier l'intégrité de votre
base de données à la fin de vos opérations :

```ruby Test de l'intégrité de la BDD
class VeryComplexeMigrationWithDestructiveTransformations < ActiveRecord::Migration
  def up
    # Some complicated operations on relations

    check_database_integrity!
  end

  def down
    raise ActiveRecord::IrreversibleMigration, "Can't recover the deleted data!"
  end

  protected

  def check_database_integrity!
    # Example test for integrity
    if OneModel.count != AnotherModel.count
      raise ActiveRecord::Rollback.new "Integrity problem"
    end
  end
end
```

## Tester ses migrations avant de déployer

Une erreur fréquente lorsque l'on développe en rails est que l'on introduit
généralement nos migrations au début du développement de la fonctionnalité :
le code de l'application possède encore à ce moment là diverses classes
qui rapidement ne feront plus parties de l'application.

Un classique est lorsque l'on change le nom d'une relation et que l'on
effectue du traitement sur ces données à l'aide du modèle associé à cette
relation :

```ruby Migration en utilisant un modèle, the wrong way
class SomeMigration < ActiveRecord::Migration
  def up
    Post.update_all(published: true)
    rename_table(:posts, :notes)
  end

  # ...
end
```

Au début de l'implémentation, la classe ``Post`` est encore présente dans le code
de l'application : la migration s'effectue donc sans aucun problème. A la fin,
la classe Post a été migrée en Note, et donc celle-ci n'existe plus.

On obtient donc lors du déploiement une jolie erreur :
`NameError: uninitialized constant Post`

Pour éviter cela, il y a deux moyens:

* [Redéfinir la classe au sein de la migration](http://blog.makandra.com/2010/03/how-to-use-models-in-your-migrations-without-killing-kittens/) :
```ruby Migration en utilisant un modèle, the right way
class SomeMigration < ActiveRecord::Migration
  class Post < ActiveRecord::Base; end

  def up
    Post.update_all(published: true)
    rename_table(:posts, :notes)
  end

  # ...
end
```

* Utiliser directement une requête SQL (ce qui est généralement plus safe) :
```ruby Migration sans modèle
class SomeMigration < ActiveRecord::Migration
  def up
    ActiveRecord::Base.execute("UPDATE posts SET published = 't'")
    rename_table(:posts, :notes)
  end

  # ...
end
```

L'avantage de la première méthode est que l'ORM s'occupera de construire la requête
SQL suivant le SGBD derrière. Typiquement le stockage des booléens ne se fait pas
de la même manière en SQLite et en PostgreSQL.

Si vous ne vous sentez pas l'âme d'un DBA, préférez la première méthode à la seconde.

## Conclusion

Une seule chose : lors de vos *code reviews*, ne négligez pas les migrations,
si celles-ci sont un peu complexes, faites les tourner sur votre machine
et n'hésitez pas à vérifier l'intégrité de votre base de données de développment.

[Loïc](https://twitter.com/loicdelmaire)
