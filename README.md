# Abstract Syntax Tree "AST", votre ami pour manipuler votre code

## Introduction
Dans cet article nous allons voir comment parcourir votre code via un AST pour y lire le contenu et chercher des patterns. Nous prendrons un exemple avec du HTML et un autre exemple avec du JavaScript. Nous serons dans un environnement Node.js dans ces exemples. Il est possible de générer des AST dans d'autres langages/environnements comme PHP, Go, Python, ...

J'utiliserais de nombreux anglicismes. Voici un rapide glossaire :
> [AST, Abstract Syntax Tree](https://fr.wikipedia.org/wiki/Arbre_de_la_syntaxe_abstraite) : un arbre dont les nœuds internes sont marqués par des opérateurs et dont les feuilles (ou nœuds externes) représentent les opérandes de ces opérateurs.

> [Parser](https://fr.wiktionary.org/wiki/parser) : outil ou action d'analyse du contenu d'un texte ou d'un fichier pour vérifier sa syntaxe ou en extraire des éléments.

## Pourquoi ?
Personnellement c'était pour de la migration de code. J'utilisais des expressions régulières de plus en plus complexes pour parser le code du projet. Ca restait limité avec toujours des cas à la marge que l'expression régulière n'attrapait pas. Ne souhaitant pas réinventer la roue, ne pas devoir interpréter le code JavaScript et ses bloques d'accolades `{}`, j'ai découvert les parsers AST qui ont grandement simplifiés et consolidés ce processus de recherche dans le code.

## Qu'est-ce que l'abstract syntax tree ?
Votre code est rempli de conditions, variables, fonctions, ... et tout cela est interprété pour être exécuter. L'AST va permettre d'avoir une arborescence de tout votre code et de le parcourir tel un objet/JSON. Chaque langage à son/ses interpréteurs.
En JavaScript, il est possible d'utiliser le parser [TypeScript](https://github.com/Microsoft/TypeScript) ou [Acorn](https://github.com/acornjs/acorn) par exemple. Le site [AST explorer](https://astexplorer.net/) permet de tester/choisir parmi différents langages et parser, puis de voir le résultat de l'AST.

Prenons par exemple ce code JavaScript :
```js
const  greetings  = {fr:  'Bonjour', en:  'Hello', jp:  'こんにちは'};
function  sayHello(name,  lang  =  'en') {
  console.log(`${greetings[lang]}  ${name} !`);
}
sayHello('Baboulinet');
```

Ce qui nous donne en AST via le parser TypeScript ([lien externe](https://astexplorer.net/#/gist/314f976d04f5cc98398292aac174cfcd/18b28c7257557b34f08b8ef38e454722ee3a8a11)) :

<iframe  width="100%"  height="600px"  src="https://astexplorer.net/#/gist/314f976d04f5cc98398292aac174cfcd/18b28c7257557b34f08b8ef38e454722ee3a8a11"></iframe>

Dans l'AST nous remarquons dans `statements` trois éléments :
- `VariableStatement` qui correspond à notre variable `greetings`
- `FunctionDeclaration` qui correspond à notre fonction `sayHello`
- `ExpressionStatement` qui correspond au lancement de notre fonction `sayHello()`

Pour chacun de ces éléments, nous avons les propriétés `pos` et `end`. Ces propriétés correspondent au numéro de caractère de début et de fin dans le fichier. C'est utile pour récupérer les portions de texte qui nous intéressent. Nous avons aussi la propriété `name.escapedText` pour avoir le nom de l'élément.
Selon le parser, le nommage de ces éléments, propriétés et la structure même de l'AST vont varier. Cependant l'idée reste la même.

## Les expressions régulières, ça aide bien aussi
Bien que l'AST découpe entièrement le code, un peu d'expressions régulières nous aidera aussi. Voici quelques exemples :
**TODO**

## Exemple HTML : Rechercher tous les `<input>` et `<textarea>` qui n'ont pas d'attribut `id`

Dans cet exemple nous allons utiliser :
- [node:path resolve](https://nodejs.org/api/path.html#pathresolvepaths) pour gérer le chemin des fichiers à parser ;
- [node:fs readFileSync](https://nodejs.org/api/fs.html#fsreadfilesyncpath-options) pour lire le contenu texte des fichiers de manière synchrone ;
- [node:util promisify](https://nodejs.org/api/util.html#utilpromisifyoriginal) pour transformer les fonctions avec callback en promise ;
- [htmlparser2](https://github.com/fb55/htmlparser2) notre parser AST HTML, `parseDocument` permet de générer l'arbre et `DomUtils` de requêter facilement son contenu ;
- [dom-serializer](https://github.com/cheeriojs/dom-serializer) pour faire transformer notre nœud AST en string HTML ;
- [chalk](https://github.com/chalk/chalk) pour mettre un peu de couleur dans nos `console.log` ;
- [glob](https://github.com/isaacs/node-glob) pour récupérer la liste des fichiers que nous souhaitons parser ;
 
Notre fichier JavaScript sera un module avec l'extension `.mjs` pour permettre l'utilisation de la syntaxe ESM `import` et l'utilisation d'`await` directement à la racine sans [IIFE](https://developer.mozilla.org/fr/docs/Glossary/IIFE).

```js
import { resolve } from 'node:path';
import { readFileSync } from 'node:fs';
import { promisify } from 'node:util';
import { parseDocument, DomUtils } from 'htmlparser2';
import render from 'dom-serializer';
import chalk from 'chalk';
import _glob from 'glob';
// transform callback to promise
const glob = promisify(_glob);

// where you want to scan html files
const PATH = resolve('../your-project');
const files = await glob(`${PATH}/**/*.html`);

files.forEach((file) => {
  // read the file's content
  const content = readFileSync(file, 'utf-8');
  // transform the content into object
  const dom = parseDocument(content);
  // tags names you want to scan
  const tags = ['input', 'textarea'];
  // store errors for output
  const errors = [];
  tags.forEach((tag) => {
    // use DomUtils.getElementsByTagName https://domutils.js.org/functions/getElementsByTagName.html
    const elements = DomUtils.getElementsByTagName(tag, dom);
    elements.forEach((element) => {
      // read the id attribute
      if (!element.attribs.id) {
        // use chalk for color, use render to transform node AST into html string
        errors.push(`❌ ${chalk.gray(render(element))}`);
      }
    });
  });
  if (errors.length) {
    console.log(`───${file.padEnd(100, '─')}`);
    console.log(errors.join('\n'));
  }
});
```
Prenons comme HTML test ce contenu :
```html
<input id="valid-input" aria-label="test avec id" class="form-check-input" type="checkbox">
<input aria-label="test checkbox" class="form-check-input" type="checkbox">
<textarea maxlength="500"></textarea>
```
A l’exécution via un `node scan.mjs` (avec `scan` comme étant le nom du fichier contenant le code ci dessus) :
```log
───/home/cub/example/test.html─────────────────────────────────────────────────────────────────────
❌ <input aria-label="test checkbox" class="form-check-input" type="checkbox">
❌ <textarea maxlength="500"></textarea>
```
Libre à vous ensuite de corriger les `id` manquants dans votre code.

## Exemple JavaScript

## Bonus

