```dataviewjs
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
	fileMap[page.file.path] = {name: page.file.name, uuid: generateUUID()}
	delete page.file
    const links = Object.entries(page).filter(([key , val]) => val.path != null).map(([key, val]) => {
    const item = {from: path, to: val.path, rel: key}
    return item
    })
    tripples = tripples.concat(links)
    
}
let rdf = '@prefix pkg: <https://pkg.io/>.\n@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>.\n@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>.\npkg:Node a rdfs:Class .\npkg:Edge a rdfs:Class .\n'

const nodes = Object.values(fileMap).map(v => `<urn:uuid:${v.uuid}> rdfs:label "${v.name}".\n<urn:uuid:${v.uuid}> rdf:type pkg:Node.\n`).join('')
rdf += nodes

const edges = tripples.map((t) => {
const edgeId = generateUUID()
let r = `<urn:uuid:${edgeId}> rdfs:label "${t.rel}".\n<urn:uuid:${edgeId}> rdf:type pkg:Edge.\n`

const from = fileMap[t.from]
if(!from) {
dv.span('No from ' + t.from + '\n')
}

const to = fileMap[t.to]
if(!to) {
dv.span('No from ' + t.to + '\n')
}
r += `<urn:uuid:${from ? from.uuid : ''}> <urn:uuid:${edgeId}> <urn:uuid:${to ? to.uuid : ''}>.\n`
return r
}).join('')
rdf += edges

dv.span(rdf)

