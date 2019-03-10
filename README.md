# [ndx](https://github.com/ndx-search/ndx) &middot; [![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/ndx-search/ndx/blob/master/LICENSE) [![npm version](https://img.shields.io/npm/v/ndx.svg)](https://www.npmjs.com/package/ndx) [![codecov](https://codecov.io/gh/ndx-search/ndx/branch/master/graph/badge.svg)](https://codecov.io/gh/ndx-search/ndx) [![CircleCI Status](https://circleci.com/gh/ndx-search/ndx.svg?style=shield&circle-token=:circle-token)](https://circleci.com/gh/ndx-search/ndx) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/ndx-search/ndx)

`ndx` is a collection of javascript (TypeScript) libraries for lightweight full-text indexing and searching.

## Live Demo

[Reddit Comments Search Engine](https://localvoid.github.io/ndx-demo/) - is a simple demo application that indexes
10,000 reddit comments. Demo application requires modern browser features:
[WebWorkers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) and
[IndexedDB](https://developer.mozilla.org/en/docs/Web/API/IndexedDB_API). Comments are stored in the IndexedDB,
and search engine is working in a WebWorker.

## Features

- Multiple fields full-text indexing and searching.
- Per-field score boosting.
- [BM25](https://en.wikipedia.org/wiki/Okapi_BM25) ranking function to rank matching documents. The same ranking
  function that is used by default in [Lucene](http://lucene.apache.org/core/) >= 6.0.0.
- [Trie](https://en.wikipedia.org/wiki/Trie) based dynamic
  [Inverted Index](https://en.wikipedia.org/wiki/Inverted_index).
- Configurable tokenizer and term filter.
- Free text queries with query expansion.
- Small memory footprint, optimized for mobile devices.
- Serializable index.

## Documentation

To check out docs for a legacy API `<1.0.0`, visit [ndx-compat](https://github.com/ndx-search/ndx-compat).

### High-level API

There are no high-level API that was available in `<1.0.0`. Everything is implemented as basic composable functions
to eliminate unused code from the final bundle.

### Quick Start

Install packages:

```sh
$ npm install --save ndx ndx-query
```

[ndx](https://github.com/ndx-search/ndx) package contains core data structures and functions that are used to mutate
this data structures.

[ndx-query](https://github.com/ndx-search/ndx-query) package contains a simple search query implementation that uses
[BM25](https://en.wikipedia.org/wiki/Okapi_BM25) ranking function to rank matching documents.

Each package has commonjs and es2015 modules.

#### Creating a simple Indexer with a Search function

```ts
import { createIndex, addDocumentToIndex } from "ndx";
import { query } from "ndx-query";
import { words } from "lodash";

function termFilter(term) {
  return term.toLowerCase();
}

function createDocumentIndex(fields) {
  // `createIndex()` creates an index data structure.
  // First argument specifies how many different fields we want to index.
  const index = createIndex(fields.length);
  // `fieldAccessors` is an array with functions that used to retrieve data from different fields. 
  const fieldAccessors = fields.map((f) => (doc) => doc[f.name]);
  // `fieldBoostFactors` is an array of boost factors for each field, in this example all fields will have
  // identical factors.
  const fieldBoostFactors = fields.map(() => 1);
  
  return {
    // `add()` function will add documents to the index.
    add: (doc) => {
      addDocumentToIndex(
        index,
        fieldAccessors,
        // Tokenizer is a function that breaks text into words, phrases, symbols, or other meaningful elements
        // called tokens.
        // Lodash function `words()` splits string into an array of its words, see https://lodash.com/docs/#words for
        // details.
        words,
        // Filter is a function that processes tokens and returns terms, terms are used in Inverted Index to
        // index documents.
        termFilter,
        // Document key, it can be a unique document id or a refernce to a document if you want to store all documents
        // in memory.
        doc.id,
        // Document.
        doc,
      );
    },
    // `search()` function will be used to perform queries.
    search: (q) => query(
      index,
      fieldBoostFactors,
      // BM25 ranking function constants:
      1.2,  // BM25 k1 constant, controls non-linear term frequency normalization (saturation).
      0.75, // BM25 b constant, controls to what degree document length normalizes tf values.
      words,
      termFilter,
      // Set of removed documents, in this example we don't want to support removing documents from the index,
      // so we can ignore it by specifying this set as `undefined` value.
      undefined, 
      q,
    ),
  };
}

const docs = [
  {
    "id": "1",
    "content": "Lorem ipsum dolor",
  },
  {
    "id": "2",
    "content": "Lorem ipsum",
  }
];

// Create a document index that will index `content` field.
const index = createDocumentIndex([{ name: "content" }]);

// Add documents to the index.
docs.forEach((d) => { index.add(d); });

// Perform a search query.
index.search("Lorem");
// => [{ key: "2" , score: ... }, { key: "1", score: ... } ]
//
// document with an id `"2"` is ranked higher because it has a `"content"` field with a less number of terms than
// document with an id `"1"`.

index.search("dolor");
// => [{ key: "1", score: ... }]
```

#### Removing Documents from the Index

```ts
import { createIndex, addDocumentToIndex, removeDocumentFromIndex, vacuumIndex } from "ndx";
import { query } from "ndx-query";
import { words } from "lodash";

function termFilter(term) {
  return term.toLowerCase();
}

function createDocumentIndex(fields) {
  const index = createIndex(fields.length);
  const fieldAccessors = fields.map((f) => (doc) => doc[f.name]);
  const fieldBoostFactors = fields.map(() => 1);
  // To support removing from the index, we need to create a set of removed documents.
  const removed = new Set();
  
  return {
    add: (doc) => {
      addDocumentToIndex(index, fieldAccessors, words, termFilter, doc.id, doc);
    },
    // `remove()` function will remove document from the index.
    remove: (id) => {
      removeDocumentFromIndex(index, removed, id);
      // When document is removed we are just marking document id as being removed. And here we are using a simple
      // heuristic to clean up index data structure when there are more than 10 removed documents.
      if (removed.size > 10) {
        vacuumIndex(index, removed);
      }
    },
    // `removed` set should be specified as 7th argument in the `query()` function.
    search: (q) => query(index, fieldBoostFactors, 1.2, 0.75, words, termFilter, removed, q),
  };
}

const docs = [
  {
    "id": "1",
    "content": "Lorem ipsum dolor",
  },
  {
    "id": "2",
    "content": "Lorem ipsum",
  }
];

const index = createDocumentIndex([{ name: "content" }]);
docs.forEach((d) => { index.add(d); });

// Remove document with id "1".
index.remove("1");
```

### Tokenizers and Filters

`ndx` library doesn't contain any tokenizers or filters. There are other libraries that implement tokenizers, for
example [Natural](https://github.com/NaturalNode/natural/) has a good collection of tokenizers and stemmers,
[stopword](https://github.com/fergiemcdowall/stopword) library can be used to filter stop words.

### Index Serialization

In some use cases it can be useful to serialize index. There are two simple functions that convert `Index` data
structure to serializable object and from serializable object:

```ts
function toSerializable<T extends SerializableDocumentKey>(index: Index<T>): SerializableIndex<T>;
function fromSerializable<T extends SerializableDocumentKey>(index: SerializableIndex<T>): Index<T>;
```

This functions are only working with indexes that are using serializable keys when indexing documents:
`string` or `number`.

#### Example

```ts
import { createIndex } from "ndx";
import { toSerializable, fromSerializable } from "ndx-serializable";

// Create index.
const index = createIndex(1);
// ... index some documents.

// Convert to data structure optimized for serialization.
const serializable = toSerializable(index);
// Encode to JSON, MsgPack or other format.
const data = JSON.stringify(serializable);

// Decode from JSON, MsgPack or other format and convert from serializable.
const index2 = fromSerializable(JSON.parse(data));
```

### Data Structures

#### `Index`

`Index` data structure is a part of a public API, there won't be any future changes to this data structure. It can be
used to create custom search query algorithms instead of the one that is implemented in the `ndx-query` package.

```ts
export interface Index<T> {
  readonly docs: Map<T, DocumentDetails<T>>;
  readonly root: InvertedIndexNode<T>;
  readonly fields: FieldDetails[];
}
```

Almost all interfaces in the `ndx` package are using parametric type `T` to specify document key type. Document key type
can be a unique document ID, or a direct reference to a document.

`docs` contains a hash map that indexes `DocumentDetails` by document key.

`root` is a root node for inverted index. Inverted index implemented as a [trie](https://en.wikipedia.org/wiki/Trie)
data structure. Inverted index keys contain terms, keys are indexed by term `charCode`. Each trie node contains a list
of documents and term frequencies for each separate field. There is always a root node that has `charCode === 0` to
simplify all algorithms that traverse through this trie.

`fields` contains additional information about each indexed field.

#### `DocumentDetails`

```ts
export interface DocumentDetails<T> {
  readonly key: T;
  readonly fieldLengths: number[];
}
```

`key` is a document key that were used to index a document. Document key type can be a unique document ID, or a direct
reference to a document.

`fieldLengths` contains a number of terms that were indexed in each field.

#### `FieldDetails`

`FieldDetails` contains additional information about each indexed field.

```ts
export interface FieldDetails {
  sum: number;
  avg: number;
}
```

`sum` is a sum of all terms that were indexed for this field.

`avg` is a an average number of terms per document for this field.

#### `InvertedIndexNode`

`InvertedIndexNode` is an inverted index [trie](https://en.wikipedia.org/wiki/Trie) node.

```ts
export interface InvertedIndexNode<T> {
  readonly charCode: number;
  next: InvertedIndexNode<T> | null;
  firstChild: InvertedIndexNode<T> | null;
  firstDoc: DocumentPointer<T> | null;
}
```

`charCode` stores indexed char codes for terms.

`next` is a reference to the next sibling node. Children lists are implemented as an intrusive singular linked lists.

`firstChild` is a reference to first children node in an intrusive singular linked list.

`firstDoc` is a reference to first document that contains a term associated with the current key. It is also implemented
as an intrusive singular linked list.

#### `DocumentPointer`

`DocumentPointer` contains term frequency for each indexed document.

```ts
export interface DocumentPointer<T> {
  next: DocumentPointer<T> | null;
  readonly details: DocumentDetails<T>;
  readonly termFrequency: number[];
}
```

`next` is a reference to the next sibling node. Document lists are implemented as an intrusive singular linked lists.

`details` is a reference to the `DocumentDetails` object that contains all details about current document.

`termFrequency` is a term frequency for each separate field.

## License

[MIT](http://opensource.org/licenses/MIT)
