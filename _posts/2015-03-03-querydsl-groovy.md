---
layout: post
title: QueryDSL (SQL) & Groovy, au diable les ORM ?!
excerpt: "Article au titre accrocheur présentant simplement un exemple d'utilisation de QueryDSL avec Groovy."
tags: [groovy, querydsl, orm, java, opinion]
comments: true
---

Depuis quelque temps, je jure au jour le jour sur cette "abstraction qui fuie" que sont les ORM. Comme d'habitude lorsque je veux paraître crédible et sérieux, je vais poser la carte Pokemon Martin Fowler et son article bien équilibré sur les ORM "Orm Hate".[^1]

Quelle alternative lorsque l'on est coincé avec une base de données relationnelle ? Le retour aux sources bien sûr, le SQL ! Avec du JDBC pur et dur ?! Meeh, il doit y avoir mieux pour ce retour à la terre, ces cultures sans pesticides, cet artisanat bio, mettons donc un peu de gouano de chauve-souris là dessus... à savoir QueryDSL et Groovy !

### QueryDSL, du SQL pour la JVM
QueryDSL est bizarrement plutôt connu pour son utilisation avec JPA. Dans notre cas on va l'utiliser pour écrire du SQL. On génère une classe Q* par table que l'on veut exploiter, ceci à l'aide d'une tâche ant/maven (ou gradle, dans ce cas il faut appeler la tache ant). Cette classe Q* contient toutes les metadata nécessaires à l'écriture de requêtes.

Par exemple, mappons une table blog(id, titre), nous générons la classe QBlog, et pouvons ainsi écrire toute sorte de requête avec l'api de QueryDSL :

{% highlight groovy %}
QBlog BLOG = QBlog.blog
Blog fetchBlog(int id) {
    SQLQuery sql = queryDslJdbcTemplate.newSqlQuery().from(BLOG).where(BLOG.id.eq(id))
    queryDslJdbcTemplate.queryForObject(sql, 
    	ConstructorExpression.create(Blog, BLOG.id, BLOG.title))
}
{% endhighlight %}

Plutôt sympathique d'avoir du sql compilé, non ? On peut écrire toute sorte de SQL tordu : sous requêtes, unions... on peut aussi utiliser un "dialecte" SQL spécifique à une BDD, voire rajouter ses propres opérateurs non standards. La classe Blog est notre classe du domaine métier, voyons voir comment on la peuple.

### Mapping vers des Pojo
Bien évidemment c'est là où les ORM sont intéressants et où la solution "tout à la main" peut devenir barbante. Cependant Groovy va nous aider à fluidifier cette étape.
Moultes solutions sont possibles :

* générer des beans en plus des Q*, ce qui permet à QueryDSL de renvoyer directement les Pojo peuplés. Je ne suis pas fan de cette approche qui impose une relation 1-1 entre Bean et Q et enlève toute liberté, ce qui est un peu le but recherché de la présente émancipation. 

* Utiliser une ConstructorExpression, comme on l'a fait dans l'exemple précédent. 
QueryDSL va utiliser la refléxion Java pour trouver un constructeur correspondant à ce que l'on appelle. En Groovy on peut utiliser l'annotation `@Canonical` pour générer un constructeur à partir des propriétés. On voit que c'est très concis, mais assez fragile car les erreurs éventuelles ne seront pas détectées à la compilation.

* Utiliser les classes `MappingProjection` de QueryDSL ou `RowMapper` de spring-data

* Utiliser l'api QueryDSL collections [^2] ou toute api de manipulation de listes d'objets, pourquoi pas les Streams de java 8 [^3] ?

* Enfin, et c'est ma préférence, on peut simplement remmener des `Tuple`, simple objet conteneur d'une ligne de résultat et traiter ce résultat avec "some Groovyness".

{% highlight groovy linenos %}
List<Blog> fetchAllBlogsWithTags() {
        SQLQuery sql = queryDslJdbcTemplate.newSqlQuery()
                            .from(BLOG)
                            .leftJoin(BLOG_TAG).on(BLOG.id.eq(BLOG_TAG.idBlog))
                            .leftJoin(TAG).on(BLOG_TAG.idTag.eq(TAG.id))
        List<Tuple> tuples = queryDslJdbcTemplate.query(sql, 
        	new QTuple(BLOG.id, BLOG.title, TAG.name))
        tuples.groupBy blogId() collect blogWithTags()
    }

    private Closure blogId() {
        return { Tuple t -> t.get(BLOG.id) }
    }

    private Closure blogWithTags() {
        return { int id, List<Tuple> ts ->
                def title = ts[0].get(BLOG.title)
                def tags = ts*.get(TAG.name)
                new Blog(id, title, tags)
        }
    }
{% endhighlight %}

- l1 à l7 : requête pour récupérer les tags en même temps que les blogs, ça peut être intéressant pour ceux et celles qui n'ont jamais vu de left join de leur vie :]
- l8 : le groupBy produit une map associant id et liste de tuples, le collect renvoie ensuite une liste de blog avec leurs tags. La fonction `blogWithTags()` tire partie du "spread dot operator" pour récupérer le tag de chaque tuple d'un seul coup.

### I'm not crazy, my creator had me tested !
Là on va oublier les tests unitaires, à part éventuellement pour les méthodes de groupage vues précédement. Cependant, (ou encore mieux j'ai envie de dire) on peut faire des tests d'intégration à moindre frais grâce à l'ami Spring :

{% highlight groovy %}
@RunWith(SpringJUnit4ClassRunner)
@SpringApplicationConfiguration(classes = DemoQuerydslApplication)
@Transactional
@Sql("/test-blogs.sql")
class DemoQuerydslApplicationTests {

	@Autowired
	BlogRepository blogRepository

	@Test
	void testFetchBlogWithId() {
		def blog = blogRepository.fetchBlog(1)
		assert blog.title == 'Titre 1'
	}
}
{% endhighlight %}

- On injecte les données de test (test-blogs.sql) en base dans une nouvelle transaction
- On teste dans le contexte de la transaction
- Spring rollback la transaction pour nous, aucun nettoyage à faire !

### Conclusion
Le projet complet est disponible sur mon github[^4] et peut servir de starter pour qui veut démarrer sur cette voie. Je conseillerais cette approche aux équipes plus à l'aise en SQL qu'avec les subtilités et la complexité des ORM. Comme d'habitude dans notre métier, on a pas THE solution à tous nos problèmes, mais une nouvelle corde à notre arc, on peut par exemple combiner ORM pour l'update/insert et SQL brut pour la partie lecture nécessitant souvent de remmener des informations spécifiques à chaque écran de l'application.

### Références
[^1]: <http://martinfowler.com/bliki/OrmHate.html>
[^2]: <https://github.com/querydsl/querydsl/tree/master/querydsl-collections>
[^3]: <http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html>
[^4]: <https://github.com/alextu/demo-querydsl>



