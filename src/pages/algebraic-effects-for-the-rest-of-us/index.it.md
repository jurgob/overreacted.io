---
title: Algebraic Effects for the Rest of Us
date: '2019-07-21'
spoiler: They’re not burritos.
cta: 'react'
---

Hai mai sentito parlare degli *algebraic effects*?

I miei primi tentativi di capire cosa siano e perche' dovrebbero interessarmi sono stati un fallimento . Ho trovato [qualche](https://www.eff-lang.org/handlers-tutorial.pdf) [pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/08/algeff-tr-2016-v2.pdf) ma non hanno fatto altro che confondermi ulteriormente. (C'e' qualcosa nei pdf accademici che mi fa' venire sonno.)

Ma il mio collega Sebastian [continuava](https://mobile.twitter.com/sebmarkbage/status/763792452289343490) [a](https://mobile.twitter.com/sebmarkbage/status/776883429400915968) [parlane](https://mobile.twitter.com/sebmarkbage/status/776840575207116800) [definendoli](https://mobile.twitter.com/sebmarkbage/status/969279885276454912) come un modello mentale per aclune delle cose che facciamo all'interno di React. (Sebastian lavora nel React team ed e' stato il promotore di diverse idee, compresi gli Hooks e il Suspence.) Ad un certo punto sono diventati un tormentone nel React team, con molte conversazioni che finivano con:

!["Algebraic Effects" caption on the "Ancient Aliens" guy meme](./effects.jpg)

Sembra che alla fine gli algebraic effects sono un concetto eccezionale, e non sono cosi' spaventosi come qui pdf mi avevano fatto pensare. **Se vuoi semplicamente usare React, puoi anche non saperne niente — ma se sei curioso, come lo ero io, continua a leggere.**

*(Disclaimer: il mio lavoro non e' fare ricerca sui linguaggi di programmazione, quindi potrei non essere stato accurato nella mia spiegazione. Non sono una autorita' in questo campo per cui fatemi sapere cosa ne pensate!)*


### Non sono ancora production ready

*Algebraic Effects* sono una funzionalita' dei linguaggi di programmazione ancora in fase di ricerca. questo vuol dire che  **a differenza dell' `if`, functions, o perfino di `async / await`, probabilmente per ora non puoi ancora usarli in produzione .** Sono supportati solo da [pochi](https://www.eff-lang.org/) [linguaggi](https://www.microsoft.com/en-us/research/project/koka/) che furono creati specificatamente per testarne l'idea. C'e' stato un progresso nel cercare di rendeli production-ready in OCaml ma e' ancora... [in corso](https://github.com/ocaml-multicore/ocaml-multicore/wiki). Il altre parole, [Can’t Touch This](https://www.youtube.com/watch?v=otCpCn0l4Wo).

>Edit: alcuni hanno citato il fatto che i linguaggi LISP [offrono qualcosa di simile](#learn-more), quindi se usi LISP, puoi usarli in produzione.

### Quindi, perche' dovrebbe importamene?

Immaginati che stai scrivendo codice usanto il  `goto`, e che qualcuno ti mostri i costrutti `if` e `for`. O che mentre sei profonamente incasinato nel callback hell, qualcuno ti mostri `async / await`. Figo, no?

Se sei il tipo di persona a cui piace imparare nuove idee sulal programmazione alcuni anni prima che diventino mainstream, potrebbe essere il momente buono per interessargli agli algebraic effects. D'altronde non devi pensare che sia *assolutamente necessario*. Sarebbe un po' come interessarsi di `async / await` nel 1999.

### Okay, Cosa sono gli Algebraic Effects?

Magari il nome puo' essere intimidatorio, ma l'idea e' veramente semplice .Se conosci il costrutto `try / catch` capirai gli algebraic effects molto velocemente.

Per prima cosa ricapitoliamo  come funziona il`try / catch`. Diciamo che hai una funzione che lancia un errore. Magari ci sono un po' di funzioni tra questa ed il primo `catch` block:

```jsx{4,19}
function getName(user) {
  let name = user.name;
  if (name === null) {
  	throw new Error('A girl has no name');
  }
  return name;
}

function makeFriends(user1, user2) {
  user1.friendNames.push(getName(user2));
  user2.friendNames.push(getName(user1));
}

const arya = { name: null, friendNames: [] };
const gendry = { name: 'Gendry', friendNames: [] };
try {
  makeFriends(arya, gendry);
} catch (err) {
  console.log("Oops, that didn't work out: ", err);
}
```

chiamiamo `throw`  dentro `getName`, ma “risale” fino a `makeFriends` raggiungendo il primo `catch`. Questa e' una proprieta' molto importante di `try / catch`. **Le cose nel mezzo non devono gestire l'errore.**

A differenza dei codice d'errori di linguaggi come C, con `try / catch`, non devi passare gli errori manualmente tra ogni livello intermedio, nella paura di perderli per strada. Vengono propagati automaticamente.

### Ma cosa c'entra questo con gli Algebraic Effects?

Nell'esempio sopra, una volta che catturiamo l'errore, non possiamo continuare. Una volta nel blocco del `catch`, non c'e modo di continare ad eseguire il codice originale.

Siamo fregati. E' troppo tardi. La cosa migliore che possiamo fare e' recuperare dal fallimento e forse riprovare in qualche modo a fare quello che stavamo facendo, ma non possiamo “andara indietro” magicamente dove eravamo e fare qualcosa di differente. **Ma con gli algebraic effects, *possiamo*.**

Questo e' un esempio scritto in un ipotetico dialetto di JavaScript (chiamiamolo ES2025 tanto per divertimento) che ci permetto di fare *recover* da un `user.name`mancante:

```jsx{4,19-21}
function getName(user) {
  let name = user.name;
  if (name === null) {
  	name = perform 'ask_name';
  }
  return name;
}

function makeFriends(user1, user2) {
  user1.friendNames.push(getName(user2));
  user2.friendNames.push(getName(user1));
}

const arya = { name: null, friendNames: [] };
const gendry = { name: 'Gendry', friendNames: [] };
try {
  makeFriends(arya, gendry);
} handle (effect) {
  if (effect === 'ask_name') {
  	resume with 'Arya Stark';
  }
}
```

*(Mi scuso con tutti i lettori del 2025 che cercano sul web “ES2025” e trovano questo articolo. Se gli algebraic effects faranno parte di JavaScript per allora, Saro' felice di aggiornarlo!)*

Invece di lanciare un'eccezione con `throw`, usiamo una ipotetica parola chiave `perform`. In maniera del tutto simile a `try / catch`, usiamo un ipotetico `try / handle`. **L'esatta sintassi non importa molto in questo contesto — Ho semplicemente inventato qualcosa per illustrare l'idea.**

Quindi cosa sta' succedendo? Diamo un occhio piu' da vicino.

Invece di lanciare un'eccezione, facciamo il *perform di un effect*. Esattamente come possiamo "lanciaro" con `throw` ogni tipo di valore, Allo stesso modo possiamo passare qualsiasi tipo di valore a `perform`. In quste esempio sto' passando una stringa, ma potrebbe essere un oggetto o qualunque altro tipo di dato:

```jsx{4}
function getName(user) {
  let name = user.name;
  if (name === null) {
  	name = perform 'ask_name';
  }
  return name;
}
```

quando lanciamo con `throw` un errore, l'engine cerca il piu' vicono `try / catch` per gestire l errore risalendo il call stack. Nello stesso modo, "performiamo" un effetto con `perform`, l'engine risalira' il call stack per trovare il piu' vicono `try / handle` per gestire l effetto:

```jsx{3}
try {
  makeFriends(arya, gendry);
} handle (effect) {
  if (effect === 'ask_name') {
  	resume with 'Arya Stark';
  }
}
```

Questo effect ci permetto di decidere come gestire il caso in cui manchi il name. La novita' qui (in confronto alle eccezzioni) e' l'ipotetico `resume with`:

```jsx{5}
try {
  makeFriends(arya, gendry);
} handle (effect) {
  if (effect === 'ask_name') {
  	resume with 'Arya Stark';
  }
}
```

Questa e' la parte che non puoi fare con `try / catch`. Ci permette di **tornare indietro a dove abbiamo perfomato l'effetto, E passareci qualcosa dall'handler delle'effetto**. 🤯

```jsx{4,6,16,18}
function getName(user) {
  let name = user.name;
  if (name === null) {
  	// 1. We perform an effect here
  	name = perform 'ask_name';
  	// 4. ...and end up back here (name is now 'Arya Stark')
  }
  return name;
}

// ...

try {
  makeFriends(arya, gendry);
} handle (effect) {
  // 2. We jump to the handler (like try/catch)
  if (effect === 'ask_name') {
  	// 3. However, we can resume with a value (unlike try/catch!)
  	resume with 'Arya Stark';
  }
}
```

Abituarsi all idea richiede un po' di tempo, ma concettualmente non e' molto differente da un “resumable `try / catch`”.

Noda ad ogni modo, che gli **algebraic effects sono molto piu' flessibili del `try / catch`, ed gli errori recoverablee' solo uno dei possibili casi.** Ho iniziato con questo semplicemente perche' e' il piu' immediato da comprendere.

### Una Funzione Non Ha Colore

Gli Algebraic effects hanno interessanti implicazioni sul codice asincrono.

In linguaggi con `async / await`, [le funzioni solitamente hanno un “colore”](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/). Ad esempio, in JavaScript non possiamo semplicemente far diventare `getName` asincrona senza “infettare” anche `makeFriends` ed il suo chiamante marcandoli con `async`. Questo puo' essere una vera spina nel fianco se *un pezzo di codice a volte deve essere sincrono, altre volte deve essere asincrono*.

```jsx
// Se vogliamo rendere questo asincrono...
async getName(user) {
  // ...
}

// Allora anche questo deve essere reso asincrono...
async function makeFriends(user1, user2) {
  user1.friendNames.push(await getName(user2));
  user2.friendNames.push(await getName(user1));
}

// E cosi' via...
```

I JavaScript generators sono [simili](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*): se stai lavorando con i generators, le cose nel mezzo devono essere al corrente dei generators.

Quindi, come fa' questo ad essere rilevante?

Per un momento, dimentichiamoci di `async / await` e torniamo al nosto esempio:

```jsx{4,19-21}
function getName(user) {
  let name = user.name;
  if (name === null) {
  	name = perform 'ask_name';
  }
  return name;
}

function makeFriends(user1, user2) {
  user1.friendNames.push(getName(user2));
  user2.friendNames.push(getName(user1));
}

const arya = { name: null, friendNames: [] };
const gendry = { name: 'Gendry', friendNames: [] };
try {
  makeFriends(arya, gendry);
} handle (effect) {
  if (effect === 'ask_name') {
  	resume with 'Arya Stark';
  }
}
```

What if our effect handler didn’t know the “fallback name” synchronously? What if we wanted to fetch it from a database?

It turns out, we can call `resume with` asynchronously from our effect handler without making any changes to `getName` or `makeFriends`:

```jsx{19-23}
function getName(user) {
  let name = user.name;
  if (name === null) {
  	name = perform 'ask_name';
  }
  return name;
}

function makeFriends(user1, user2) {
  user1.friendNames.push(getName(user2));
  user2.friendNames.push(getName(user1));
}

const arya = { name: null, friendNames: [] };
const gendry = { name: 'Gendry', friendNames: [] };
try {
  makeFriends(arya, gendry);
} handle (effect) {
  if (effect === 'ask_name') {
  	setTimeout(() => {
      resume with 'Arya Stark';
  	}, 1000);
  }
}
```

In this example, we don’t call `resume with` until a second later. You can think of `resume with` as a callback which you may only call once. (You can also impress your friends by calling it a “one-shot delimited continuation.”)

Now the mechanics of algebraic effects should be a bit clearer. When we `throw` an error, the JavaScript engine “unwinds the stack”, destroying local variables in the process. However, when we `perform` an effect, our hypothetical engine would *create a callback* with the rest of our function, and `resume with` calls it.

**Again, a reminder: the concrete syntax and specific keywords are made up for this article. They’re not the point, the point is in the mechanics.**

### A Note on Purity

It’s worth noting that algebraic effects came out of functional programming research. Some of the problems they solve are unique to pure functional programming. For example, in languages that *don’t* allow arbitrary side effects (like Haskell), you have to use concepts like Monads to wire effects through your program. If you ever read a Monad tutorial, you know they’re a bit tricky to think about. Algebraic effects help do something similar with less ceremony.

This is why so much discussion about algebraic effects is incomprehensible to me. (I [don’t know](/things-i-dont-know-as-of-2018/) Haskell and friends.) However, I do think that even in an impure language like JavaScript, **algebraic effects can be a very powerful instrument to separate the *what* from the *how* in the code.**

They let you write code that focuses on *what* you’re doing:

```jsx{2,3,5,7,12}
function enumerateFiles(dir) {
  const contents = perform OpenDirectory(dir);
  perform Log('Enumerating files in ', dir);
  for (let file of contents.files) {
  	perform HandleFile(file);
  }
  perform Log('Enumerating subdirectories in ', dir);
  for (let directory of contents.dir) {
  	// We can use recursion or call other functions with effects
  	enumerateFiles(directory);
  }
  perform Log('Done');
}
```

And later wrap it with something that specifies *how*:

```jsx{6-7,9-11,13-14}
let files = [];
try {
  enumerateFiles('C:\\');
} handle (effect) {
  if (effect instanceof Log) {
  	myLoggingLibrary.log(effect.message);
  	resume;
  } else if (effect instanceof OpenDirectory) {
  	myFileSystemImpl.openDir(effect.dirName, (contents) => {
      resume with contents;
  	});
  } else if (effect instanceof HandleFile) {
    files.push(effect.fileName);
    resume;
  }
}
// The `files` array now has all the files
```

Which means that those pieces can even become librarified:

```jsx
import { withMyLoggingLibrary } from 'my-log';
import { withMyFileSystem } from 'my-fs';

function ourProgram() {
  enumerateFiles('C:\\');
}

withMyLoggingLibrary(() => {
  withMyFileSystem(() => {
    ourProgram();
  });
});
```

Unlike `async / await` or Generators, **algebraic effects don’t require complicating functions “in the middle”**. Our `enumerateFiles` call could be deep within `ourProgram`, but as long as there’s an effect handler *somewhere above* for each of the effects it may perform, our code would still work.

Effect handlers let us decouple the program logic from its concrete effect implementations without too much ceremony or boilerplate code. For example, we could completely override the behavior in tests to use a fake filesystem and to snapshot logs instead of outputting them to the console:

```jsx{19-23}
import { withFakeFileSystem } from 'fake-fs';

function withLogSnapshot(fn) {
  let logs = [];
  try {
  	fn();
  } handle (effect) {
  	if (effect instanceof Log) {
  	  logs.push(effect.message);
  	  resume;
  	}
  }
  // Snapshot emitted logs.
  expect(logs).toMatchSnapshot();
}

test('my program', () => {
  const fakeFiles = [/* ... */];
  withFakeFileSystem(fakeFiles, () => {
  	withLogSnapshot(() => {
	  ourProgram();
  	});
  });
});
```

Because there is no [“function color”](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) (code in the middle doesn’t need to be aware of effects) and effect handlers are *composable* (you can nest them), you can create very expressive abstractions with them.

### A Note on Types

Because algebraic effects are coming from statically typed languages, much of the debate about them centers on the ways they can be expressed in types. This is no doubt important but can also make it challenging to grasp the concept. That’s why this article doesn’t talk about types at all. However, I should note that usually the fact that a function can perform an effect would be encoded into its type signature. So you shouldn’t end up in a situation where random effects are happening and you can’t trace where they’re coming from.

You might argue that algebraic effects technically do [“give color”](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) to functions in statically typed languages because effects are a part of the type signature. That’s true. However, fixing a type annotation for an intermediate function to include a new effect is not by itself a semantic change — unlike adding `async` or turning a function into a generator. Inference can also help avoid cascading changes. An important difference is you can “bottle up” an effect by providing a noop or a mock implementation (for example, a sync call for an async effect), which lets you prevent it from reaching the outer code if necessary — or turn it into a different effect.

### Should We Add Algebraic Effects to JavaScript?

Honestly, I don’t know. They are very powerful, and you can make an argument that they might be *too* powerful for a language like JavaScript.

I think they could be a great fit for a language where mutation is uncommon, and where the standard library fully embraced effects. If you primarily do `perform Timeout(1000)`, `perform Fetch('http://google.com')`, and `perform ReadFile('file.txt')`, and your language has pattern matching and static typing for effects, it might be a very nice programming environment.

Maybe that language could even compile to JavaScript!

### How Is All of This Relevant to React?

Not that much. You can even say it’s a stretch.

If you watched [my talk about Time Slicing and Suspense](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html), the second part involves components reading data from a cache:

```jsx
function MovieDetails({ id }) {
  // What if it's still being fetched?
  const movie = movieCache.read(id);
}
```

*(The talk uses a slightly different API but that’s not the point.)*

This builds on a React feature called “Suspense”, which is in active development for the data fetching use case. The interesting part, of course, is that the data might not yet be in the `movieCache` — in which case we need to do *something* because we can’t proceed below. Technically, in that case the `read()` call throws a Promise (yes, *throws* a Promise — let that sink in). This “suspends” the execution. React catches that Promise, and remembers to retry rendering the component tree after the thrown Promise resolves.

This isn’t an algebraic effect per se, even though this trick was [inspired](https://mobile.twitter.com/sebmarkbage/status/941214259505119232) by them. But it achieves the same goal: some code below in the call stack yields to something above in the call stack (React, in this case) without all the intermediate functions necessarily knowing about it or being “poisoned” by `async` or generators. Of course, we can’t really *resume* execution in JavaScript later, but from React’s point of view, re-rendering a component tree when the Promise resolves is pretty much the same thing. You can cheat when your programming model [assumes idempotence](/react-as-a-ui-runtime/#purity)!

[Hooks](https://reactjs.org/docs/hooks-intro.html) are another example that might remind you of algebraic effects. One of the first questions that people ask is: how can a `useState` call possibly know which component it refers to?

```jsx
function LikeButton() {
  // How does useState know which component it's in?
  const [isLiked, setIsLiked] = useState(false);
}
```

I already explained the answer [near the end of this article](/how-does-setstate-know-what-to-do/): there is a “current dispatcher” mutable state on the React object which points to the implementation you’re using right now (such as the one in `react-dom`). There is similarly a “current component” property that points to our `LikeButton`’s internal data structure. That’s how `useState` knows what to do.

Before people get used to it, they often think it’s a bit “dirty” for an obvious reason. It doesn’t “feel right” to rely on shared mutable state. *(Side note: how do you think `try / catch` is implemented in a JavaScript engine?)*

However, conceptually you can think of `useState()` as of being a `perform State()` effect which is handled by React when executing your component. That would “explain” why React (the thing calling your component) can provide state to it (it’s above in the call stack, so it can provide the effect handler). Indeed, [implementing state](https://github.com/ocamllabs/ocaml-effects-tutorial/#2-effectful-computations-in-a-pure-setting) is one of the most common examples in the algebraic effect tutorials I’ve encountered.

Again, of course, that’s not how React *actually* works because we don’t have algebraic effects in JavaScript. Instead, there is a hidden field where we keep the current component, as well as a field that points to the current “dispatcher” with the `useState` implementation. As a performance optimization, there are even separate `useState` implementations [for mounts and updates](https://github.com/facebook/react/blob/2c4d61e1022ae383dd11fe237f6df8451e6f0310/packages/react-reconciler/src/ReactFiberHooks.js#L1260-L1290). But if you squint at this code very hard, you might see them as essentially effect handlers.

To sum up, in JavaScript, throwing can serve as a crude approximation for IO effects (as long as it’s safe to re-execute the code later, and as long as it’s not CPU-bound), and having a mutable “dispatcher” field that’s restored in `try / finally` can serve as a crude approximation for synchronous effect handlers.

You can also get a much higher fidelity effect implementation [with generators](https://dev.to/yelouafi/algebraic-effects-in-javascript-part-4---implementing-algebraic-effects-and-handlers-2703) but that means you’ll have to give up on the “transparent” nature of JavaScript functions and you’ll have to make everything a generator. Which is... yeah.

### Learn More

Personally, I was surprised by how much algebraic effects made sense to me. I always struggled understanding abstract concepts like Monads, but Algebraic Effects just “clicked”. I hope this article will help them “click” for you too.

I don’t know if they’re ever going to reach mainstream adoption. I think I’ll be disappointed if they don’t catch on in any mainstream language by 2025. Remind me to check back in five years!

I’m sure there’s so much more you can do with them — but it’s really difficult to get a sense of their power without actually writing code this way. If this post made you curious, here’s a few more resources you might want to check out:

* https://github.com/ocamllabs/ocaml-effects-tutorial

* https://www.janestreet.com/tech-talks/effective-programming/

* https://www.youtube.com/watch?v=hrBq8R_kxI0

Many people also pointed out that if you omit the typing aspects (as I did in this article), you can find much earlier prior art for this in the [condition system](https://en.wikibooks.org/wiki/Common_Lisp/Advanced_topics/Condition_System) in Common Lisp. You might also enjoy reading James Long’s [post on continuations](https://jlongster.com/Whats-in-a-Continuation) that explains how the `call/cc` primitive can also serve as a foundation for building resumable exceptions in userland.

If you find other useful resources on algebraic effects for people with JavaScript background, please let me know on Twitter!
