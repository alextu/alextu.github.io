---
layout: post
title: Angular 2 et les "databinding" de la discorde
excerpt: "Cet homme est tombé sur un article traitant des databinding dans angular 2, vous n'allez pas en croire vos yeux !"
tags: [angularjs, javascript, montagejs, databinding, opinion]
comments: true
---

L'article en question [^1] est certes écrit par l'auteur d'un framework conccurent et ancien d'angular, cependant le code ne ment pas :


{% highlight javascript %}

this.contact = new Contact('John', 'Doe');  
this.contactForm = new FormControlGroup("form", [
      new FormControl("firstName"),
      new FormControl("lastName")
	]);
this.contactForm.readFrom(this.contact);

{% endhighlight %}

Il semblerait en effet que l'équipe d'angular ait décidé d'abandonner les "two-way databinding" (les ng-model en fait), pour des raisons de performances à priori. On voit qu'ils sont partiellement "remplacés" par des objets FormControl intermédiaires. Notez que je n'ai rien contre les FormControl, j'ai moi même un ami FormControl alors c'est dire... m'enfin quand même ! Dupliquer la structure du formulaire dans le code js du ViewModel, yerk !

Angular 1 est certe partie sur de mauvaises bases en autorisant uniquement le "two-way databinding", mais était-ce une raison pour en prendre l'antithèse ? Où est donc la synthèse dirait Egel ou votre prof de français ?

### The others

Beaucoup de technos ont toujours offert les deux possibilités (one-way, two-way) d'une manière ou d'une autre : Flex, Cocoa, WebObjects pour citer les moins célèbres. Dans le monde javascript il y en a sans doute une floppée (remarquez que cette phrase générique est valable pour tout sujet en javascript). Je vais citer le framework que je trouve le plus clair sur ce point : MontageJS. Celui-ci utilise la lib "Functional Reactive Bindings" [^2] écrite par Kris Kowal l'auteur des "promises" ($q dans angular), c'est donc prometteur ! -_-

{% highlight javascript %}

var object = {};
var cancel = bind(object, "foo", {"<->": "bar"});

// <-
object.bar = 10;
expect(object.foo).toBe(10);

// ->
object.foo = 20;
expect(object.bar).toBe(20);

{% endhighlight %}

 Comme dirait l'oncle Bob, du "clean code", c'est du code qui exprime clairement ce que l'auteur conçoit. En l'occurence, qu'est-ce qu'un databinding ? Une synchronisation entre une propriété d'un objet et une autre propriété d'un objet.
 "<->" pour du "two-way databinding", "<-" pour du "one-way databinding". MontageJS est orienté composant dès le départ avec une très forte séparation des responsabilités, cette déclaration de bindings se fait non pas en js, mais en déclaratif en en tête du template :

 {% highlight javascript %}

	{
	    "text": {
	        "prototype": "montage/ui/text.reel",
	        "properties": {
	            "element": {"#": "text"}
	        },
	        "bindings": {
	            "value": {"<-": "@owner.nom"}
	        }
	    }
	}

{% endhighlight %}

Un exemple de "one-way databinding" : la valeur du composant texte sera synchronisée avec la proriété "nom" du ViewModel et pas inversement.

Angular 2 semble séduisant sur d'autres aspects en éliminant pas mal de code obscur d'angular 1. Ce serait dommage d'en rajouter, l'"utilisabilité" d'un framework est un critère tout aussi important que la performance. Espérons qu'ils revoient leur copie. [^3]


### Références
[^1]: http://blog.durandal.io/2015/03/17/aurelia-angular-2-0-code-side-by-side-part-2/
[^2]: https://github.com/montagejs/frb/blob/master/README.md
[^3]: https://docs.google.com/document/d/1US9h0ORqBltl71TlEU6s76ix8SUnOLE2jabHVg9xxEA/edit