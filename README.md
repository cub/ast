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
const greetings = {fr: 'Bonjour', en: 'Hello', jp: 'こんにちは'};
function sayHello(name, lang = 'en') {
  console.log(`${greetings[lang]} ${name} !`);
}
sayHello('Baboulinet');
```

Ce qui nous donne en AST via le parser TypeScript ([lien externe](https://astexplorer.net/#/gist/314f976d04f5cc98398292aac174cfcd/18b28c7257557b34f08b8ef38e454722ee3a8a11)) :

<iframe width="100%" height="600px" src="https://astexplorer.net/#/gist/314f976d04f5cc98398292aac174cfcd/18b28c7257557b34f08b8ef38e454722ee3a8a11"></iframe>

Dans l'AST nous remarquons dans `statements` trois éléments :
- `VariableStatement` qui correspond à notre variable `greetings`
- `FunctionDeclaration` qui correspond à notre fonction `sayHello`
- `ExpressionStatement` qui correspond au lancement de notre fonction `sayHello()`

Pour chacun de ces éléments, nous avons les propriétés `pos` et `end`. Ces propriétés correspondent au numéro de caractère de début et de fin dans le fichier. C'est utile pour récupérer les portions de texte qui nous intéressent. Nous avons aussi la propriété `name.escapedText` pour avoir le nom de l'élément.
Selon le parser, le nommage de ces éléments, propriétés et la structure même de l'AST vont varier. Cependant l'idée reste la même.

## Exemples : 

Dans les exemples ci dessous, nous allons utiliser différents librairies :

- [node:path resolve](https://nodejs.org/api/path.html#pathresolvepaths) pour gérer le chemin des fichiers à parser ;
- [node:fs readFileSync](https://nodejs.org/api/fs.html#fsreadfilesyncpath-options) pour lire le contenu texte des fichiers de manière synchrone ;
- [node:fs writeFileSync](https://nodejs.org/api/fs.html#fswritefilesyncfile-data-options) pour écrire le contenu texte des fichiers de manière synchrone ;
- [node:util promisify](https://nodejs.org/api/util.html#utilpromisifyoriginal) pour transformer les fonctions avec callback en promise ;
- [htmlparser2](https://github.com/fb55/htmlparser2) notre parser AST HTML, `parseDocument` permet de générer l'arbre et `DomUtils` de requêter facilement son contenu ;
- [dom-serializer](https://github.com/cheeriojs/dom-serializer) pour faire transformer notre nœud AST provenant de `htmlparser2` en string HTML ;
- [typescript](https://github.com/microsoft/TypeScript) notre parser AST JavaScript (compatible évidemment aussi avec TypeScript) ;
- [chalk](https://github.com/chalk/chalk) pour mettre un peu de couleur dans nos `console.log` ;
- [glob](https://github.com/isaacs/node-glob) pour récupérer la liste des fichiers que nous souhaitons parser ;
 
Nos fichiers JavaScript seront des modules avec l'extension `.mjs` pour permettre l'utilisation de la syntaxe ESM `import` et l'utilisation d'`await` directement à la racine sans [IIFE](https://developer.mozilla.org/fr/docs/Glossary/IIFE).

On notera les paramètres du `parseDocument(xxx, { recognizeSelfClosing: true, lowerCaseTags: false })` et du `render(xxx, { encodeEntities: false, xmlMode: true, selfClosingTags: true })` pour garder la mise en forme des déclarations des composants.

Exemple sans ces paramètres :
```html
<MonComposant :id="v$.toto" />
<! deviendrait -->
<moncomposant :id="v&#x24;.toto"></moncomposant>
```

### [HTML] Rechercher tous les `<input>` et `<textarea>` qui n'ont pas d'attribut `id`

```js
// scan.mjs
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
const PATH = resolve('./your-project');
const files = await glob(`${PATH}/**/*.html`);

files.forEach((file) => {
  // read the file's content
  const content = readFileSync(file, 'utf-8');
  // transform the content into object
  const dom = parseDocument(content, { recognizeSelfClosing: true, lowerCaseTags: false });
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
        errors.push(`❌ ${chalk.gray(render(element, { encodeEntities: false, xmlMode: true, selfClosingTags: true }))}`);
      }
    });
  });
  if (errors.length) {
    console.log(`───${file.padEnd(100, '─')}`);
    console.log(errors.join('\n'));
  } else {
    console.log('✅ everything is ok');
  }
});
```
Prenons comme HTML test ce contenu :
```html
<!-- example.html -->
<input id="valid-input" aria-label="test avec id" class="form-check-input" type="checkbox">
<input aria-label="test checkbox" class="form-check-input" type="checkbox">
<textarea maxlength="500"></textarea>
```
A l’exécution via un `node scan.mjs` (avec `scan.mjs` comme étant le nom du fichier contenant le code JavaScript ci-dessus), nous aurons comme output :
```log
───/home/cub/example.html─────────────────────────────────────────────────────────────────────
❌ <input aria-label="test checkbox" class="form-check-input" type="checkbox">
❌ <textarea maxlength="500"></textarea>
```
Libre à vous ensuite de corriger les `id` manquants dans votre code.

### [HTML + JavaScript] Migrer un fichier .vue de la syntaxe OptionsAPI en syntaxe CompositionAPI

Nous allons prendre comme exemple ce contenu [https://vuejs.org/examples/#hello-world](https://vuejs.org/examples/#hello-world). Dans cette page il est possible de changer l'apercu de syntaxe en haut à gauche. Pour l'exemple nous avons rajouté un data `cats: ['meow', 'miaou']`.

```html
<!-- example.vue -->
<script>
export default {
  data() {
    return {
      message: 'Hello World!',
      cats: ['meow', 'miaou']
    }
  }
}
</script>

<template>
  <h1>{{ message }}</h1>
</template>
```
Le but est de récupérer dans l'objet Vue, son `data`, puis toutes ses propriétés et leurs valeurs pour les réécrire en `const xxx = ref(yyy)`.
Pour connaitre les chemins des différents éléments à lire dans l'AST, le site [https://astexplorer.net](https://astexplorer.net) est bien utile. Il est possible sinon de mettre des points d'arret dans votre IDE et de lancer votre script en debug pour visionner en live le contenu de votre AST ([voir tutoriel pour VS Code](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)).

```js
// options-to-composition.mjs
import { resolve } from 'node:path';
import { readFileSync, writeFileSync } from 'node:fs';
import { promisify } from 'node:util';
import { parseDocument, DomUtils } from 'htmlparser2';
import ts from 'typescript';
import render from 'dom-serializer';
import _glob from 'glob';
// transform callback to promise
const glob = promisify(_glob);
// where you want to scan html files
const PATH = resolve('./your-project');
const files = await glob(`${PATH}/**/*.vue`);

files.forEach((file) => {
  // read the file's content
  const content = readFileSync(file, 'utf-8');
  // transform the content into object
  const dom = parseDocument(content, { recognizeSelfClosing: true, lowerCaseTags: false });
  // stock the <script> content
  const script = render(DomUtils.getElementsByTagName('script', dom)[0].children, { encodeEntities: false, xmlMode: true, selfClosingTags: true });
  // stock the <template>
  const template = render(DomUtils.getElementsByTagName('template', dom)[0], { encodeEntities: false, xmlMode: true, selfClosingTags: true });
  // the AST for JavaScript (the content of script)
  const node = ts.createSourceFile('x.ts', script, ts.ScriptTarget.Latest);

  // all properties of the OptionsApi (data, computed, methods, ...)
  const rootProperties = node.statements[0].expression.properties;
  // search for the data node
  const dataNode = rootProperties.find((prop) => prop.name.escapedText === 'data');
  if (!dataNode) { console.log('❌ no optionsApi.data found'); return; }
  // get all properties declared inside data
  const dataProperties = dataNode.body.statements[0].expression.properties;
  // rewrite to Ref for CompositionApi
  // each types (text, array, ...) have differents accessors
  // ex "message: 'Hello World!'" >> prop.initializer.text
  // ex "cats: ['meow', 'miaou']" >> prop.initializer.elements[]
  // we use pos & end to get raw text without care about types
  const dataRefs = dataProperties.map((prop) => `const ${prop.name.escapedText} = ref(${script.slice(prop.initializer.pos, prop.initializer.end).trim()});`);
  // rewrite whole .vue file
  const composition = `<script setup>
import { ref } from 'vue'

${dataRefs.join('\n')}
</script>

${template}`;
  // writing the output to a different file with "-composition" suffix
  writeFileSync(file.replace('.vue', '-composition.vue'), composition, 'utf-8');
  console.log(`✅ migrated ${file}`);
});
```

Ce qui aura pour effet de créer un nouveau fichier `example-composition.vue` ayant comme contenu :
```html
<script setup>
import { ref } from 'vue'

const message = ref('Hello World!');
const cats = ref(['meow', 'miaou']);
</script>

<template>
  <h1>{{ message }}</h1>
</template>
```
Bien sûr ce script ne couvre que très partiellement une réécriture de la syntaxe OptionsAPI en CompositionAPI. Il faudrait gérer les `computed`, les `props`, ... Et il faudrait surtout repenser votre code pour l'organiser de manière cohérente avec la CompositionAPI.

## Conclusion

J'espère que cela vous a inspiré à créer des scripts pour checker/manipuler votre code !
