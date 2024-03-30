It is a demo repository for article on medium about Personal Knowledge Graphs and Obsidian 
https://volodymyrpavlyshyn.medium.com/how-to-export-your-obsidian-vault-to-rdf-00fb2539ed18



## Linked Data and RDF Triplets

Linked Data and RDF (Resource Description Framework) triples are closely related concepts aimed at representing structured data and relationships in a machine-readable way for the Semantic Web. Let’s discuss these concepts and how they relate to each other.

**RDF (Resource Description Framework):**

RDF is a W3C standard for modeling and representing information about resources on the web. It provides a flexible and extensible way to describe relationships between resources using simple, subject-predicate-object statements called triples. RDF is one of the core technologies enabling the Semantic Web, as it allows applications to merge, query, and consume data easily.

**RDF Triples**

An RDF triple is the fundamental building block of RDF data, representing a single statement about a resource in the form of a subject-predicate-object structure:

![](https://miro.medium.com/v2/resize:fit:862/0*kxXBMF12Gj4rUuXh.png)

RDF triple

1.  Subject: The resource that the statement is about. A URI usually represents it.
2.  Predicate: The property or relationship that connects the subject to the object. A URI also represents it.
3.  Object: The value or resource the subject relates to via the predicate. It can be either a URI or a literal value.

RDF triples can be serialized in various formats, such as RDF/XML, Turtle, N-Triples, and JSON-LD. Each design has its way of representing the subject-predicate-object structure, but they all convey the same information.

So, we need to generate unique uuid identifiers for every object. In RDF, there is no difference between nodes and edges, but we want to differentiate the types of entries.

Let's create a simple ontology.

```
@prefix pkg: https://pkg.io/.
@prefix rdf: http://www.w3.org/1999/02/22-rdf-syntax-ns#.
@prefix rdfs: http://www.w3.org/2000/01/rdf-schema#.
pkg:Node a rdfs:Class .
pkg:Edge a rdfs:Class .
```

We have types of Node and Edge.

Node representation is a pair of triples that include a type and label. We will use page names as node labels

```
<urn:uuid:6549cc7e-e605-4b93-9039-c09be6bf004f> rdfs:label "Personal Knowledge Graph".
<urn:uuid:6549cc7e-e605-4b93-9039-c09be6bf004f> rdf:type pkg:Node.
```

Edges are the same we have a thee of triples:

-   type
-   label
-   triple relation

```
<urn:uuid:792e475b-d503-4e3d-a785-2f71f4f510f8> rdfs:label "linked".
<urn:uuid:792e475b-d503-4e3d-a785-2f71f4f510f8> rdf:type pkg:Edge.
<urn:uuid:c003d751-9604-4a1c-b893-0cd38d75fa67> <urn:uuid:792e475b-d503-4e3d-a785-2f71f4f510f8> <urn:uuid:55b2f738-087a-46a2-bf34-338ce07b44df>.

```

We create a new Link every time. Another possible implementation is to create a dictionary of link labels and represent them as one resource.

## Enable JS queries in a dataview

To do a complex script we need to use js API and javascript queries in a data view. It is disabled by default.

Lets enable it

![](https://miro.medium.com/v2/resize:fit:1400/1*FH1uvBoOkjgc-9-Xp7Bzcw.png)

## Let's Jump to a code.

As I mention we need to use UUID as identifier so lets make naive implementation of it

```js
function generateUUID() { // Public Domain/MIT
    var d = new Date().getTime();//Timestamp
    var d2 = ((typeof performance !== 'undefined') && performance.now && (performance.now()*1000)) || 0;//Time in microseconds since page-load or 0 if unsupported
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
        var r = Math.random() * 16;//random number between 0 and 16
        if(d > 0){//Use timestamp until depleted
            r = (d + r)%16 | 0;
            d = Math.floor(d/16);
        } else {//Use microseconds since page-load if supported
            r = (d2 + r)%16 | 0;
            d2 = Math.floor(d2/16);
        }
        return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
    });
}
```

Now we will scan all pages is your vault

```js
const pages = dv.pages()
```

For simple obsidian links that are not typed links, we just need to scan outgoing links and create a map of it

```js

```

For named links, we should get attributes and properties of a page that has a value as links

```js
const outgoingLinks =[... new Set(page.file.outlinks.values.map(l => l.path))]
```

Now we need to turn a link to a triples

```js
Object.entries(page).filter(([key , val]) => val.path != null)
```

we don't want to process links twice so we expect that we filter out named links from ongoing links set

Now we ready to create a Nodes entries

```
let rdf = '@prefix pkg: <https://pkg.io/>.\n@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>.\n@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>.\npkg:Node a rdfs:Class .\npkg:Edge a rdfs:Class .\n'

const nodes = Object.values(fileMap).map(v => `<urn:uuid:${v.uuid}> rdfs:label "${v.name}".\n<urn:uuid:${v.uuid}> rdf:type pkg:Node.\n`).join('')
rdf += nodes
```

Now we will figure out that not all page links has a page in obsidian so we need to takes care and collect such pages

```js
const edges = tripples.map((t) => {
const edgeId = generateUUID()
let r = `<urn:uuid:${edgeId}> rdfs:label "${t.rel}".\n<urn:uuid:${edgeId}> rdf:type pkg:Edge.\n`

const from = fileMap[t.from]
if(!from) {
  if (!missedPages[t.from]) {
   missedPages[t.from] = generateUUID()
  }
}

const to = fileMap[t.to]
if(!to) {
     if (!missedPages[t.to]) {
   missedPages[t.to] = generateUUID()
  }
}
r += `<urn:uuid:${from ? from.uuid : missedPages[t.from]}> <urn:uuid:${edgeId}> <urn:uuid:${to ? to.uuid :   missedPages[t.to]}>.\n`
return r
}).join('')
```

So now we could create a missed node that has no pages

```js
const missedNodes = Object.entries(missedPages).map(([key, val]) => `<urn:uuid:${val}> rdfs:label "${key}".\n<urn:uuid:${val}> rdf:type pkg:Node.\n`).join('')
rdf += missedNodes
```

Let's render a result.

Missed page links

```js
dv.header(2, 'Missed Pages')
dv.list(Object.keys(missedPages).map(p => `[[${p}]]`))
```

Now user could do a simple click and create a pages

finally we could generate our RDF

```js
dv.header(2,'Rdf file')
dv.span(rdf)
```

## Final script

```js
function generateUUID() { // Public Domain/MIT
    var d = new Date().getTime();//Timestamp
    var d2 = ((typeof performance !== 'undefined') && performance.now && (performance.now()*1000)) || 0;//Time in microseconds since page-load or 0 if unsupported
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
        var r = Math.random() * 16;//random number between 0 and 16
        if(d > 0){//Use timestamp until depleted
            r = (d + r)%16 | 0;
            d = Math.floor(d/16);
        } else {//Use microseconds since page-load if supported
            r = (d2 + r)%16 | 0;
            d2 = Math.floor(d2/16);
        }
        return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
    });
}
const pages = dv.pages()

const fileMap = {}
let tripples = []
for (let p = 0 ; p < pages.length ; p++) {
 const page = pages[p]
 const path = page.file.path
 const outgoingLinks =[... new Set(page.file.outlinks.values.map(l => l.path))]
 const processedLinks = {}
 fileMap[page.file.path] = {name: page.file.name, uuid: generateUUID()}
 delete page.file
    const links = Object.entries(page).filter(([key , val]) => val.path != null).map(([key, val]) => {
    const item = {from: path, to: val.path, rel: key}
    processedLinks[val.path] = 1
    return item
    })
    tripples = tripples.concat(links)
    const unnamedLinks = outgoingLinks.filter(l => !processedLinks[l] ).map(l => ({from: path , to: l , rel: 'linked'}))
    tripples = tripples.concat(unnamedLinks)
    
}
let rdf = '@prefix pkg: <https://pkg.io/>.\n@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>.\n@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>.\npkg:Node a rdfs:Class .\npkg:Edge a rdfs:Class .\n'

const nodes = Object.values(fileMap).map(v => `<urn:uuid:${v.uuid}> rdfs:label "${v.name}".\n<urn:uuid:${v.uuid}> rdf:type pkg:Node.\n`).join('')
rdf += nodes

const missedPages = {}

const edges = tripples.map((t) => {
const edgeId = generateUUID()
let r = `<urn:uuid:${edgeId}> rdfs:label "${t.rel}".\n<urn:uuid:${edgeId}> rdf:type pkg:Edge.\n`

const from = fileMap[t.from]
if(!from) {
  if (!missedPages[t.from]) {
   missedPages[t.from] = generateUUID()
  }
}

const to = fileMap[t.to]
if(!to) {
     if (!missedPages[t.to]) {
   missedPages[t.to] = generateUUID()
  }
}
r += `<urn:uuid:${from ? from.uuid : missedPages[t.from]}> <urn:uuid:${edgeId}> <urn:uuid:${to ? to.uuid :   missedPages[t.to]}>.\n`
return r
}).join('')

const missedNodes = Object.entries(missedPages).map(([key, val]) => `<urn:uuid:${val}> rdfs:label "${key}".\n<urn:uuid:${val}> rdf:type pkg:Node.\n`).join('')
rdf += missedNodes

rdf += edges
dv.header(2, 'Missed Pages')
dv.list(Object.keys(missedPages).map(p => `[[${p}]]`))
dv.header(2,'Rdf file')
dv.span(rdf)
```

![](https://miro.medium.com/v2/resize:fit:1400/1*gAFrTklYMEHjzQOCbTRNgQ.png)

RDF sample

```rdf
@prefix pkg: https://pkg.io/.
@prefix rdf: http://www.w3.org/1999/02/22-rdf-syntax-ns#.
@prefix rdfs: http://www.w3.org/2000/01/rdf-schema#.
pkg:Node a rdfs:Class .
pkg:Edge a rdfs:Class .
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> rdfs:label "Personal Knowledge Graph".
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> rdf:type pkg:Node.
<urn:uuid:43dad791-b00e-40ec-9ada-2e45a2881d93> rdfs:label "User centric".
<urn:uuid:43dad791-b00e-40ec-9ada-2e45a2881d93> rdf:type pkg:Node.
<urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb> rdfs:label "Knowledge Graph".
<urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb> rdf:type pkg:Node.
<urn:uuid:6b4cc8bc-8654–4a5c-8604-a9699fa07241> rdfs:label "Edges".
<urn:uuid:6b4cc8bc-8654–4a5c-8604-a9699fa07241> rdf:type pkg:Node.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> rdfs:label "me".
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> rdf:type pkg:Node.
<urn:uuid:5ac18eb8–6d66–4794-b93e-2304d24cc908> rdfs:label "User data".
<urn:uuid:5ac18eb8–6d66–4794-b93e-2304d24cc908> rdf:type pkg:Node.
<urn:uuid:1d28ad39-eebd-4e97–9e71–4f0f9e6fd141> rdfs:label "RDF".
<urn:uuid:1d28ad39-eebd-4e97–9e71–4f0f9e6fd141> rdf:type pkg:Node.
<urn:uuid:038eaa65–7170–4541–9a10-ff00938bb3ba> rdfs:label "test".
<urn:uuid:038eaa65–7170–4541–9a10-ff00938bb3ba> rdf:type pkg:Node.
<urn:uuid:128329c8–4347–4fcd-b123-aea10b6ce1a0> rdfs:label "boom".
<urn:uuid:128329c8–4347–4fcd-b123-aea10b6ce1a0> rdf:type pkg:Node.
<urn:uuid:55b2f738–087a-46a2-bf34–338ce07b44df> rdfs:label "bam".
<urn:uuid:55b2f738–087a-46a2-bf34–338ce07b44df> rdf:type pkg:Node.
<urn:uuid:358665eb-2e57–433a-9c01-cc83354ac792> rdfs:label "Nodes".
<urn:uuid:358665eb-2e57–433a-9c01-cc83354ac792> rdf:type pkg:Node.
<urn:uuid:f7a87cea-e12a-4fff-a5b8–4f8f75b5a9e1> rdfs:label "Metadata".
<urn:uuid:f7a87cea-e12a-4fff-a5b8–4f8f75b5a9e1> rdf:type pkg:Node.
<urn:uuid:a7f6adca-a798–40e3-ba6d-56a80193b3e3> rdfs:label "is".
<urn:uuid:a7f6adca-a798–40e3-ba6d-56a80193b3e3> rdf:type pkg:Edge.
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> <urn:uuid:a7f6adca-a798–40e3-ba6d-56a80193b3e3> <urn:uuid:43dad791-b00e-40ec-9ada-2e45a2881d93>.
<urn:uuid:61a30b72–2b3e-4ccb-8646–21122efbc013> rdfs:label "kind".
<urn:uuid:61a30b72–2b3e-4ccb-8646–21122efbc013> rdf:type pkg:Edge.
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> <urn:uuid:61a30b72–2b3e-4ccb-8646–21122efbc013> <urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb>.
<urn:uuid:fb4486da-8a06–4570-aa29-ee6fdd1e0c94> rdfs:label "about".
<urn:uuid:fb4486da-8a06–4570-aa29-ee6fdd1e0c94> rdf:type pkg:Edge.
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> <urn:uuid:fb4486da-8a06–4570-aa29-ee6fdd1e0c94> <urn:uuid:5ac18eb8–6d66–4794-b93e-2304d24cc908>.
<urn:uuid:583d3489-de0c-4239–9657–417a18bb0465> rdfs:label "linked".
<urn:uuid:583d3489-de0c-4239–9657–417a18bb0465> rdf:type pkg:Edge.
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> <urn:uuid:583d3489-de0c-4239–9657–417a18bb0465> <urn:uuid:128329c8–4347–4fcd-b123-aea10b6ce1a0>.
<urn:uuid:7bc02fc5-a329–43dd-a1f6-cbfd2525f99f> rdfs:label "linked".
<urn:uuid:7bc02fc5-a329–43dd-a1f6-cbfd2525f99f> rdf:type pkg:Edge.
<urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f> <urn:uuid:7bc02fc5-a329–43dd-a1f6-cbfd2525f99f> <urn:uuid:55b2f738–087a-46a2-bf34–338ce07b44df>.
<urn:uuid:df7dc30b-823a-4529–9169-ecc760b24ed1> rdfs:label "linked".
<urn:uuid:df7dc30b-823a-4529–9169-ecc760b24ed1> rdf:type pkg:Edge.
<urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb> <urn:uuid:df7dc30b-823a-4529–9169-ecc760b24ed1> <urn:uuid:358665eb-2e57–433a-9c01-cc83354ac792>.
<urn:uuid:b09453c6-cfe2–4a36-bf12-c862ed7c61bd> rdfs:label "linked".
<urn:uuid:b09453c6-cfe2–4a36-bf12-c862ed7c61bd> rdf:type pkg:Edge.
<urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb> <urn:uuid:b09453c6-cfe2–4a36-bf12-c862ed7c61bd> <urn:uuid:6b4cc8bc-8654–4a5c-8604-a9699fa07241>.
<urn:uuid:50a0d38f-2ecc-422d-b08e-29931539d773> rdfs:label "linked".
<urn:uuid:50a0d38f-2ecc-422d-b08e-29931539d773> rdf:type pkg:Edge.
<urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb> <urn:uuid:50a0d38f-2ecc-422d-b08e-29931539d773> <urn:uuid:f7a87cea-e12a-4fff-a5b8–4f8f75b5a9e1>.
<urn:uuid:f6b77600–2fe3–4b59–86ac-17c2f620cfe5> rdfs:label "poitTo".
<urn:uuid:f6b77600–2fe3–4b59–86ac-17c2f620cfe5> rdf:type pkg:Edge.
<urn:uuid:6b4cc8bc-8654–4a5c-8604-a9699fa07241> <urn:uuid:f6b77600–2fe3–4b59–86ac-17c2f620cfe5> <urn:uuid:358665eb-2e57–433a-9c01-cc83354ac792>.
<urn:uuid:b325d666–3135–4911-a39a-a13487d78383> rdfs:label "poitto".
<urn:uuid:b325d666–3135–4911-a39a-a13487d78383> rdf:type pkg:Edge.
<urn:uuid:6b4cc8bc-8654–4a5c-8604-a9699fa07241> <urn:uuid:b325d666–3135–4911-a39a-a13487d78383> <urn:uuid:358665eb-2e57–433a-9c01-cc83354ac792>.
<urn:uuid:f174daa2-dcc3–495d-b126-c0a39488c1d2> rdfs:label "like".
<urn:uuid:f174daa2-dcc3–495d-b126-c0a39488c1d2> rdf:type pkg:Edge.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> <urn:uuid:f174daa2-dcc3–495d-b126-c0a39488c1d2> <urn:uuid:6549cc7e-e605–4b93–9039-c09be6bf004f>.
<urn:uuid:b56fc03b-6b3b-4c70-b75d-1894b5490d76> rdfs:label "keenAbout".
<urn:uuid:b56fc03b-6b3b-4c70-b75d-1894b5490d76> rdf:type pkg:Edge.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> <urn:uuid:b56fc03b-6b3b-4c70-b75d-1894b5490d76> <urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb>.
<urn:uuid:678861ed-a4e3–4bcc-ab69-c9f376933761> rdfs:label "build".
<urn:uuid:678861ed-a4e3–4bcc-ab69-c9f376933761> rdf:type pkg:Edge.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> <urn:uuid:678861ed-a4e3–4bcc-ab69-c9f376933761> <urn:uuid:43dad791-b00e-40ec-9ada-2e45a2881d93>.
<urn:uuid:2260071a-50a0–40e5-a448-be7a60e3a83b> rdfs:label "keenabout".
<urn:uuid:2260071a-50a0–40e5-a448-be7a60e3a83b> rdf:type pkg:Edge.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> <urn:uuid:2260071a-50a0–40e5-a448-be7a60e3a83b> <urn:uuid:2964cb00–8fc2–4cd0-acb3–4fc05f9933eb>.
<urn:uuid:8bdd8ba8-abd4–4483–9ceb-a1ce926a9ecf> rdfs:label "linked".
<urn:uuid:8bdd8ba8-abd4–4483–9ceb-a1ce926a9ecf> rdf:type pkg:Edge.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> <urn:uuid:8bdd8ba8-abd4–4483–9ceb-a1ce926a9ecf> <urn:uuid:128329c8–4347–4fcd-b123-aea10b6ce1a0>.
<urn:uuid:792e475b-d503–4e3d-a785–2f71f4f510f8> rdfs:label "linked".
<urn:uuid:792e475b-d503–4e3d-a785–2f71f4f510f8> rdf:type pkg:Edge.
<urn:uuid:c003d751–9604–4a1c-b893–0cd38d75fa67> <urn:uuid:792e475b-d503–4e3d-a785–2f71f4f510f8> <urn:uuid:55b2f738–087a-46a2-bf34–338ce07b44df>.

```

You could find a test vault on my github
