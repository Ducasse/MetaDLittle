  I checked that the self-description, the library schema and the corrected data in it all parse with DLittleLParser in
  the image, using evaluation only. There's no validator yet, so the rules themselves haven't been run against anything.

  # DLittle Schema Language — Design Rules

  2026-10-27 · Stéphane Ducasse

  ## Overview

  DLittle schemas are written in DLittle itself. A schema declares which entities,
  types and nesting a DLittle document may contain, and a validator checks documents
  against it. Sixteen rules define how these declarations are written. Where the
  metadata is stored is a separate question and out of scope.

  A DLittle node is a key, one value, and a list of child nodes:

  ```
  (a:b
      (c:d (e:f) (g:h))
      (z:k (r:t) (p:q)))
  ```

  Design choice: **entity types are separate from key names.** An `entity:`
  declaration names a concept (`Book`), says which key introduces it in data
  (`title`), the type of its value, and which entities it may contain. This was
  preferred over three alternatives: a flat list of declarations per key, a typed
  copy of the data's shape, and a shape with named reusable parts. It matches the
  goal of declaring the types of entities, and it leaves room to map entities to
  Pharo classes later.

  ## Rules 1–5: structure and types

  ### Rule 1 — A value is always one atom

  **Rule.** Every value, in schemas and in data, is a single atom. A list or any
  other structure is written only as child nodes, never packed inside a value.

  **Example.**

  ```
  (entity: Book (key: title) (value: string)
      (contains: Read) (contains: Style) (contains: Price))
  ```

  and not `(contains: Read Style Price)` or `(field: read boolean)`.

  **Discussion.** `DLittleLParser` reads `(field: read boolean)` as the key `field`
  with the single string value `read boolean`, so packing several things into one
  value hides structure inside a string. Keeping values atomic keeps DLittle uniform,
  so schemas can be checked by the same machinery as data. Text such as
  `Aldiss / Brian` is still one atom: a string.

  ### Rule 2 — `occurs` defaults to `one`

  **Rule.** A `contains:` may have an optional `(occurs: …)` child saying how often
  the contained entity may appear. When it is missing, it means `one`.

  | occurs     | Meaning             |
  |------------|---------------------|
  | `one`      | exactly once (default) |
  | `optional` | zero or one         |
  | `many`     | zero or more        |
  | `some`     | one or more         |

  **Example.**

  ```
  (contains: Price)                      exactly once
  (contains: Style (occurs: optional))
  (contains: Serie (occurs: many))
  (contains: Book  (occurs: some))
  ```

  **Discussion.** Three alternatives were rejected: `min:`/`max:` children, separate
  keys such as `requires:` and `optional:`, and regex-style markers such as `Serie*`.
  The markers would also break Rule 1. A default of `one` matches XML Schema and the
  The markers would also break Rule 1. A default of `one` matches XML Schema and the
  most common case, so making something optional or repeated is always a deliberate
  choice. Ranges such as "between 2 and 5" can be added later with `min:`/`max:`
  without breaking existing schemas.

  ### Rule 3 — Closed sets of values are declared in DLittle

  **Rule.** A closed set of allowed values is declared as a `type:` with `choice:`
  children. It is not hard-coded in the validator.

  **Example.**

  ```
  (type: genre      (base: string) (choice: SF) (choice: fantaisie))
  (type: occurrence (base: string)
      (choice: one) (choice: optional) (choice: many) (choice: some))
  ```

  **Discussion.** The allowed values belong to the domain: the data uses `SF` and
  `fantaisie`. The self-description declares its own `occurrence` type with the same
  mechanism. `base:` comes from Rule 8.

  ### Rule 4 — Four built-in types

  **Rule.** The built-in types are exactly `string`, `number`, `boolean` and `name`.

  | Type      | Accepts                                              |
  |-----------|------------------------------------------------------|
  | `string`  | any atom, e.g. `Aldiss / Brian`                      |
  | `number`  | an atom that reads as a number, e.g. `1`             |
  | `boolean` | `true` or `false`; other spellings via aliases (Rule 7) |
  | `name`    | an identifier that refers to a declaration (Rule 6)  |

  **Example.** `(entity: Price (key: price) (value: number))`
  **Discussion.** The first version had three built-in types, and `boolean` was a
  declared closed set. It was revised so that `boolean` is built in with the fixed
  spellings `true`/`false`, giving a later conversion to Pharo objects a meaning it
  can rely on. Spellings such as `yes`/`no` are handled with aliases. Dates and
  decimal precision wait until a real document needs them.

  ### Rule 5 — A schema is one `schema:` node with a `root:`

  **Rule.** A schema is a single top-level `(schema: …)` node. Its `(root: …)` child
  names the entity that data documents start with, and it contains all the
  `entity:` and `type:` declarations.

  **Example.**

  ```
  (schema: library
      (root: Author)
      (entity: Author …)
      (entity: Book …)
      (type: answer …))
  ```

  **Discussion.** Without a root, a validator cannot know what is allowed at the top
  of a document. Wrapping the declarations makes a schema one ordinary DLittle node
  rather than a bare sequence of nodes. The self-description is itself a schema, so
  it starts `(schema: dlittle (root: Schema) …)`: every schema document starts with a
  `schema:` node, so its starting entity is `Schema`.

  ## Rules 6–11: references, types and consistency

  ### Rule 6 — Every `name` value must resolve

  **Rule.** When a schema is checked, every value of type `name` must refer to an
  existing declaration. `contains:` and `root:` must name a declared entity;
  `value:` and `base:` must name a built-in type or a declared type.

  **Example.** This is rejected because no entity `Bok` exists:

  ```
  (entity: Serie (key: serie) (value: string) (contains: Bok (occurs: some)))
  ```

  **Discussion.** Catching references is the reason `name` exists as a type. A typo
  is reported once, when the schema is checked, rather than showing up later as
  confusing errors on valid data.

  ### Rule 7 — Aliases apply within one type

  **Rule.** A type may declare aliases, which are other spellings for its values,
  as `(alias: X (for: Y))`. An alias applies only to values of that type.

  **Example.**

  ```
  (type: answer (base: boolean)
      (alias: yes (for: true))
      (alias: no  (for: false)))
  (type: genre (base: string) (choice: SF) (choice: fantaisie)
      (alias: science-fiction (for: SF)))
  (entity: Read (key: read) (value: answer))
  ```

  **Discussion.** A schema-wide alias would make `yes` mean `true` everywhere, even
  inside a `string` such as a title. Keeping each alias inside its type keeps its
  meaning local. Aliases replaced an earlier idea of adding `(means: true)` to each
  choice.

  ### Rule 8 — Every type has exactly one `base:`

  **Rule.** Every `type:` has exactly one `(base: …)`. `choice:` and `alias:` are
  optional and can repeat.

  **Example.**

  ```
  (type: rating (base: number) (choice: 1) (choice: 2) (choice: 3))
  ```

  **Discussion.** Without a mandatory base, a type would need "a base, or some
  choices, or both", and the schema language has no way to say either/or. With a
  mandatory base, the self-description can still describe types, choices always
  narrow down a base, and every declared type leads back to a built-in type when
  values are checked.

  ### Rule 9 — No chaining of types

  **Rule.** `base:` may only name one of the four built-in types, never a declared
  type.

  **Example.** This is rejected:

  ```
  (type: frenchAnswer (base: answer) (alias: oui (for: true)))
  ```

  **Discussion.** Chaining would raise further questions: whether aliases and choices
  are inherited, how choices can be narrowed, and what happens with cycles. With this
  rule, every type is one step from a built-in type. Chaining can be allowed later
  without breaking existing schemas.

  ### Rule 10 — Choices and aliases must be consistent

  **Rule.** Checking a schema rejects a type if a `choice:` is not a valid value of
  its base, or if an alias's `for:` is not a valid value of the type. A valid value
  of the type is one of its choices if it has any; otherwise it is any value of its
  base.

  **Example.** All three are rejected:

  ```
  (type: rating (base: number)  (choice: 1) (choice: two))
  (type: genre  (base: string)  (choice: SF) (alias: scifi (for: SciFi)))
  (type: answer (base: boolean) (alias: yes (for: maybe)))
  ```

  **Discussion.** This follows the same idea as Rule 6: catch mistakes in the
  schema, not in the data. Because of Rule 9, the check only needs to look at the
  type itself.

  ### Rule 11 — Keys are unique among an entity's children

  **Rule.** Different entities may share a key, but checking a schema rejects an
  entity whose `contains:` lists two entities with the same key.

  **Example.** This is allowed, because the two `name` entities never have the same
  parent:

  ```
  (entity: AuthorName    (key: name) (value: string))
  (entity: PublisherName (key: name) (value: string))
  (entity: Author    (key: author)    (contains: AuthorName))
  (entity: Publisher (key: publisher) (contains: PublisherName))
  ```

  **Discussion.** To find the entity for a node in a data document, the validator
  looks through the entities its parent may contain for the one with a matching key.
  The same word often means different things in different places, and lookup only
  needs keys to be unique among the children of one entity.

  ## Rules 12–16: checking data

  ### Rule 12 — The order of children doesn't matter

  **Rule.** Children may appear in any order in data. Only how often each entity
  appears is checked.

  **Example.** Both are valid:

  ```
  (title: L''hiver d''helliconia (read: no) (style: SF) (price: 1))
  (title: L''hiver d''helliconia (price: 1) (read: no) (style: SF))
  ```

  **Discussion.** DLittle documents are written by hand, and order carries no
  meaning. If it ever matters, it can be added later as an opt-in setting on an
  entity.

  ### Rule 13 — Schemas are closed by default

  **Rule.** A key that the schema does not declare at that point in the data is an
  error.

  **Example.** Rejected, because `Book` declares no entity with the key `prise`:

  ```
  (title: L''hiver d''helliconia (read: no) (style: SF) (prise: 1))
  ```

  **Discussion.** The main purpose of a schema is to catch typos. An open-by-default
  schema would let `prise` through silently.

  ### Rule 14 — An entity accepts extra keys only with `(open: true)`

  **Rule.** An entity that declares `(open: true)` accepts children with keys it
  does not declare. If the flag is missing, it means `false`.

  **Example.**

  ```
  (entity: Book (key: title) (value: string)
      (open: true)
      (contains: Read) (contains: Style) (contains: Price))
  ```

  so that this is valid:

  ```
  (title: L''hiver d''helliconia (read: no) (style: SF) (price: 1) (note: signed copy))
  ```

  **Discussion.** The rejected alternative was a built-in wildcard entity such as
  `(contains: Any)`. It would need a special name that Rule 6 has to know about, and
  `occurs` makes little sense for it. The flag is a plain `boolean`, and the
  self-description only gains one small `Open` entity.

  ### Rule 15 — Extras are unchecked subtrees, kept in full

  **Rule.** An extra child can be any DLittle subtree and is not checked. When a
  document is converted to Pharo objects, extras are stored in a dictionary-like
  `properties` structure, mapping each key to its parsed DLittle node (a `DLList`).

  **Example.**

  ```
  (note: signed copy (date: 1982) (place: Paris))
  ```

  becomes `book properties at: 'note'`, a node holding the value `signed copy` and
  the children `date` and `place`. Code that only needs the text uses
  `(book properties at: 'note') value`.

  **Discussion.** Storing only the value string would silently drop children.
  Storing the parsed node keeps everything.

  ### Rule 16 — An extra key may appear at most once

  **Rule.** The validator rejects an extra key that appears more than once under the
  same entity, so `properties` always maps a key to exactly one node.

  **Example.** Rejected:

  ```
  (title: … (note: signed copy) (note: first edition))
  ```

  **Discussion.** Letting the last one win would lose data without warning. Always
  storing a collection would make the common single case awkward:
  `(… at: 'note') first value`. A key that repeats is really part of the structure,
  so it should be declared with `(occurs: many)`.

  ## The self-description

  Here is the schema language described in DLittle, using all sixteen rules:
  ```
  (schema: dlittle
      (root: Schema)
      (entity: Schema   (key: schema)   (value: name)
          (contains: Root)
          (contains: Entity (occurs: some))
          (contains: Type   (occurs: many)))
      (entity: Root     (key: root)     (value: name))
      (entity: Entity   (key: entity)   (value: name)
          (contains: Key)
          (contains: Value    (occurs: optional))
          (contains: Open     (occurs: optional))
          (contains: Contains (occurs: many)))
      (entity: Key      (key: key)      (value: name))
      (entity: Value    (key: value)    (value: name))
      (entity: Open     (key: open)     (value: boolean))
      (entity: Contains (key: contains) (value: name)
          (contains: Occurs (occurs: optional)))
      (entity: Occurs   (key: occurs)   (value: occurrence))
      (entity: Type     (key: type)     (value: name)
          (contains: Base)
          (contains: Choice (occurs: many))
          (contains: Alias  (occurs: many)))
      (entity: Base     (key: base)     (value: name))
      (entity: Choice   (key: choice)   (value: string))
      (entity: Alias    (key: alias)    (value: string)
          (contains: For))
      (entity: For      (key: for)      (value: string))
      (type: occurrence (base: string)
          (choice: one) (choice: optional) (choice: many) (choice: some)))
  ```

  `Value` is optional because DLittle allows a node with only children, such as
  `(a: (b:c))`. Once a validator exists, this schema can be checked against itself.

  ## Worked example: a library

  The schema:

  ```
  (schema: library
      (root: Author)
      (entity: Author     (key: author) (value: string)
          (contains: Serie      (occurs: many))
          (contains: Standalone (occurs: optional)))
      (entity: Serie      (key: serie)  (value: string)
          (contains: Book (occurs: some)))
      (entity: Standalone (key: books)  (value: string)
          (contains: Book (occurs: some)))
      (entity: Book       (key: title)  (value: string)
          (open: true)
          (contains: Read) (contains: Style) (contains: Price))
      (entity: Read       (key: read)   (value: answer))
      (entity: Style      (key: style)  (value: genre))
      (entity: Read       (key: read)   (value: answer))
      (entity: Style      (key: style)  (value: genre))
      (entity: Price      (key: price)  (value: number))
      (type: answer (base: boolean)
          (alias: yes (for: true)) (alias: no (for: false)))
      (type: genre (base: string)
          (choice: SF) (choice: fantaisie)))
  ```

  The data, with balanced parentheses (the original was missing three closing ones):

  ```
  (author : Aldiss / Brian
      (serie : helliconia
          (title: Le printemps d''helliconia (read: no) (style: SF) (price: 1))
          (title: Helliconia, l''ete (read: no) (style: SF) (price: 1))
          (title: L''hiver d''helliconia (read: no) (style: SF) (price: 1)))
      (books : individual
          (title: L''instant de l''eclipse (read: no) (style: fantaisie) (price: 1))))
  ```

  `Book` is shared by `Serie` and `Standalone`. Reusing one entity like this is the
  reason entity types are separate from keys.

  ## Out of scope and open questions

  - **Converting documents to Pharo objects** is postponed. Open points: how entities
    map to classes, how children become slots or collections, where aliases are
    resolved (`yes` → `true`), and writing objects back to DLittle. Only Rules 15–16
    (`properties`) are fixed so far.
  - **Where the metadata is stored** (a separate file, a header, inline): a separate
    question from the design of the declarations.
  - **Later extensions that won't break existing schemas:** `min:`/`max:` ranges
    (Rule 2), opt-in ordering (Rule 12), chaining types (Rule 9), dates and decimals
    (Rule 4).
  - **Not decided:** whether an alias may have the same spelling as one of its type's
    choices, for example `(choice: SF) (alias: SF (for: fantaisie))`.


