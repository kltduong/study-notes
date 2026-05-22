# Modern JS Syntax — Course Progress

## Calibration

**Starting point:** Working JS developer, comfortable with ES6+ in daily use. Deep mechanics covered in other courses (scope, inheritance, async). Syntax is used but not systematically studied — gaps likely in newer features (private fields, decorators, generators, proxies) and precise semantics of familiar ones.

**Goal:** Reading and writing fluency — parse any modern JS correctly, use features idiomatically. Not engine internals, but "what does this mean and how do I use it right." Like learning grammar of a language.

**Planned arc:** Class syntax → fields & visibility → inheritance syntax → destructuring → spread/rest → optional chaining & nullish coalescing → template literals → iterators & for...of → generators → symbols → proxy & reflect → misc modern syntax. Quizzes after thematic groups, tests at part boundaries.

## Progress

- [ ] Class fundamentals — `class` declaration/expression, constructor, methods, getters/setters, computed names
- [ ] Class fields & visibility — public/private fields (`#`), static fields/methods, static blocks
- [ ] Inheritance syntax — `extends`, `super` (constructor + method), method overriding, `instanceof`
- [ ] Quiz 1 (covering *Class fundamentals* through *Inheritance syntax*)
- [ ] Destructuring — arrays, objects, nested, defaults, rest in destructuring, parameter destructuring
- [ ] Spread & rest — spread in arrays/objects/calls, rest parameters, shallow copy idiom
- [ ] Optional chaining & nullish coalescing — `?.`, `??`, `??=`, difference from `&&`/`||`
- [ ] Quiz 2 (covering *Destructuring* through *Optional chaining & nullish coalescing*)
- [ ] Test 1 (part 1: covering *Class fundamentals* through *Optional chaining & nullish coalescing*)
- [ ] Template literals — tagged templates, nesting, raw strings
- [ ] Iterators & for...of — `Symbol.iterator`, iterable protocol, `for...of` vs `for...in`
- [ ] Generators — `function*`, `yield`, `yield*`, lazy sequences, generator as iterator
- [ ] Quiz 3 (covering *Template literals* through *Generators*)
- [ ] Symbols & well-known symbols — `Symbol()`, `Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.hasInstance`
- [ ] Proxy & Reflect — traps, handler, `Reflect` methods, common patterns (validation, logging, defaults)
- [ ] Misc modern syntax — shorthand properties/methods, logical assignment (`||=`, `&&=`, `??=`), `globalThis`, `Object.hasOwn`, `structuredClone`, `using` (explicit resource management)
- [ ] Quiz 4 (covering *Symbols & well-known symbols* through *Misc modern syntax*)
- [ ] Test 2 (part 2: covering *Template literals* through *Misc modern syntax*)
- [ ] Final test (cumulative)
