**The SPARQL Library of Common Core Ontologies**

The goal of this project is to develop a suite of SPARQL queries that will serve as quality control (QC) checks against the [Common Core Ontologies](https://github.com/CommonCoreOntology/CommonCoreOntologies) suite. These queries will be designed to identify and flag potential issues, ensuring the ontology's integrity, consistency, and adherence to predefined standards.

**Assignment Details**

Your task is to construct SPARQL queries to be included in the [CCO QC workflow](https://github.com/CommonCoreOntology/CommonCoreOntologies/actions). Ideally, your queries will be added to the CCO repository [here](https://github.com/CommonCoreOntology/CommonCoreOntologies/tree/develop/.github/deployment/sparql). 

Your queries will be ranked in terms of difficulty. The lowest - 8 - indicates a rather easy query, while the highest - 1 - will indicate a very sophisticated query. 

For our purposes, the more sophisticated queries will be worth more points than less sophisticated, and you are required to submit enough queries to acquire 100 points according to the following point system: 


  | **Query Sophistication** | **Points** |
  | ------------------------ | ---------- |
  |       1                  |      35    |
  |       2                  |      25    |
  |       3                  |      20    |
  |       4                  |      10    |
  |       5                  |       5    |
  |       6                  |       3    |
  |       7                  |       2    |
  |       8                  |       0    |

You're probably thinking, "why would I submit a level 8 if they're not worth any points?" Great question. Because everyone has to submit at least one level 8. Otherwise, you're permitted to submit in any distribution you choose. For example, you might submit 2 queries for level one (70 points), one for level 3 (20 points), one for 4 (10 points), and one for kata 8 (0 points but required). 

It is your responsibility and the responsibility of your peers reviewing your submission in PR to determine whether your submission is ranked appropriately. In the event that consensus is reached that your query is ranked inappropriately, you must work with your peers to revise the submission so that it is either more or less challenging, accordingly. You are not permitted to submit new queries with different strengths after PRs are open, but must instead revise your PRs. So, think hard about how challenging your submission is. 

**Template**

The SPARQL queries should have the template: 
Title
    (descriptive title of the query)
Constraint Description: 
    (description of the query functionality)
Severity:
    (select "Warning" or "Error")

Your query should end with a BIND clause and an associated ?error in the SELECT. For example: 

  - BIND (concat("WARNING: The following ontology elements have the same rdfs:label ", str(?element), " and ", str(?element2)) AS ?error)

**Guidance**

A few tips for developing effective SPARQL queries for the Common Core Ontologies (CCO):
  - Review the [existing SPARQL queries](https://github.com/CommonCoreOntology/CommonCoreOntologies/tree/develop/.github/deployment/sparql) so as not to duplicate work
  - Review [documentation and design patterns](https://github.com/CommonCoreOntology/CommonCoreOntologies/tree/develop/documentation) to understand stucture of CCO
  - Understand common issues in ontologies; [explore the OOPS!](https://oa.upm.es/35873/1/INVE_MEM_2014_192872.pdf) list here for inspiration
  - Observe annotation conventions, e.g. use of labels, comments, etc. must be present and accurate

When creating queries, start with simple quality control checks and build complexity through practice. Feel free to leverage generative AI for this project. Also, feel free to collaborate with peers. 

Be sure to test your queries. You may do this in Protege or in the [SPARQL playground](https://atomgraph.github.io/SPARQL-Playground/). 





Misuse of transitive properties: Query has inputs for given instances where transitive property is used to relate two instances indirectly but not directly. This prevents error in the reasoners, might be most helpful with HermiT, Pellet, and Fact++. Query allows for instances to be inputed, which can help identify human error in making sure all applicable instances have been labeled trasnitive or not. (lvl. 4)

PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX owl: <http://www.w3.org/2002/07/owl#>
PREFIX cco: <http://www.ontologyrepository.com/CommonCoreOntologies/Mid/AgentOntology#>

SELECT DISTINCT ?transitiveProperty ?instance1 ?instance2 ?instance3 ?error WHERE {
    ?transitiveProperty rdf:type owl:TransitiveProperty .
    ?instance1 ?transitiveProperty ?instance2 .
    ?instance2 ?transitiveProperty ?instance3 .
    FILTER NOT EXISTS { ?instance1 ?transitiveProperty ?instance3 }
    BIND (concat("ERROR: Misuse of transitive property ", str(?transitiveProperty), " detected: ", str(?instance1), " -> ", str(?instance2), " -> ", str(?instance3), " without direct relation to ", str(?instance3)) AS ?error)
}



Defining wrong symmetric relationships: query has instance inputs to plug in instances where symmetry is used and will show error if “insanceA? relates to instanceB?” but “instanceB? does not relate to instanceA?” Similar to previous but checks symmetry (lvl. 4)

PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX owl: <http://www.w3.org/2002/07/owl#>
PREFIX cco: <http://www.ontologyrepository.com/CommonCoreOntologies/Mid/AgentOntology#>

SELECT DISTINCT ?symmetricProperty ?instanceA ?instanceB ?error WHERE {
    ?symmetricProperty rdf:type owl:SymmetricProperty .
    ?instanceA ?symmetricProperty ?instanceB .
    FILTER NOT EXISTS { ?instanceB ?symmetricProperty ?instanceA }
    BIND (concat("ERROR: Symmetrical property ", str(?symmetricProperty), " is misused: ", str(?instanceA), " is related to ", str(?instanceB), " but not vice versa.") AS ?error)
}


Missing annotations: query pulls all classes without annotations. Helps to identify all classes without annotations.
(lvl 8)

SELECT ?class
WHERE {
  ?class a owl:Class .
  FILTER NOT EXISTS { ?class rdfs:label ?label }
  FILTER NOT EXISTS { ?class rdfs:comment ?comment }
  FILTER NOT EXISTS { ?class skos:definition ?definition }
  FILTER NOT EXISTS { ?class ?annotationProperty ?annotationValue .
                     ?annotationProperty a owl:AnnotationProperty }
}




-Missing domain or range in properties: pulls all instances of properties without domains and ranges. (lvl.2)


SELECT ?property ?propertyType (IF(BOUND(?domain), "Has Domain", "Missing Domain") AS ?domainStatus) 
                                (IF(BOUND(?range), "Has Range", "Missing Range") AS ?rangeStatus)
WHERE {
  # Find object or data properties
  ?property a rdf:Property .
  OPTIONAL { ?property rdfs:domain ?domain . }
  OPTIONAL { ?property rdfs:range ?range . }

  # Check if the property is an object property or data property
  OPTIONAL { ?property a owl:ObjectProperty . BIND("ObjectProperty" AS ?propertyType) }
  OPTIONAL { ?property a owl:DatatypeProperty . BIND("DatatypeProperty" AS ?propertyType) }

  # Exclude deprecated properties
  FILTER NOT EXISTS { ?property owl:deprecated true }

  # Either domain or range must be missing for the property to be included in the results
  FILTER ( !BOUND(?domain) || !BOUND(?range) )
}
ORDER BY ?property ?propertyType



-Redundant object properties: finds object properties with same range and domain. This can help identify identical ranges and domains to compare. (lvl 3)


SELECT ?property1 ?property2 ?domain ?range
WHERE {
  # Find two different object properties
  ?property1 a owl:ObjectProperty .
  ?property2 a owl:ObjectProperty .
  FILTER (?property1 != ?property2)  # Ensure they are not the same property

  # Get their domain and range
  ?property1 rdfs:domain ?domain .
  ?property1 rdfs:range ?range .
  ?property2 rdfs:domain ?domain .
  ?property2 rdfs:range ?range .

  # Exclude deprecated properties
  FILTER NOT EXISTS { ?property1 owl:deprecated true }
  FILTER NOT EXISTS { ?property2 owl:deprecated true }
}
ORDER BY ?domain ?range ?property1 ?property2



-Inconsistent property ranges: identifies properties that have different ranges when used with subclasses of a given class. (lvl3)

SELECT ?property ?superclass ?subclass1 ?subclass2 ?range1 ?range2
WHERE {
  # Identify a property and its associated domain (a superclass)
  ?property a rdf:Property .
  ?property rdfs:domain ?superclass .

  # Find two different subclasses of the superclass
  ?subclass1 rdfs:subClassOf ?superclass .
  ?subclass2 rdfs:subClassOf ?superclass .
  FILTER (?subclass1 != ?subclass2)

  # Find ranges associated with the property for each subclass
  ?subclass1 rdfs:subClassOf ?superclass .
  ?subclass2 rdfs:subClassOf ?superclass .
  
  ?subclass1 ?property ?range1 .
  ?subclass2 ?property ?range2 .

  # Ensure the ranges are different for the two subclasses
  FILTER (?range1 != ?range2)

  # Optional: Exclude deprecated properties
  FILTER NOT EXISTS { ?property owl:deprecated true }
}
ORDER BY ?property ?superclass ?subclass1 ?subclass2



Duplicate names: Finds if there are two instances that refer to the same thing (lvl7)

SELECT ?label (GROUP_CONCAT(?instance; separator=", ") AS ?instances)
WHERE {
  ?instance rdfs:label ?label .
}
GROUP BY ?label
HAVING (COUNT(?instance) > 1)
ORDER BY ?label


Query that may be able to find code that are located in annotations, this can potentially find malicious code. Query seeks to immitate syntax. (lvl 6)

SELECT ?annotationProperty ?instance ?annotation
WHERE {
  ?instance ?annotationProperty ?annotation .
  
  # Look for common code-like patterns
  FILTER regex(?annotation, "^(\\s*\\w+\\s*\\(|\\{.*\\}|\\[.*\\]|;|\\b(if|for|function|return)\\b)", "i")
