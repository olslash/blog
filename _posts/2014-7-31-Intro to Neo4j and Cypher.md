This is a quick intro to the Neo4j graph database and its Cypher query language, based on a talk I gave at Hack Reactor a few days ago. By the end of this post, you'll have a general idea of how to read Cypher, you'll have a starting point for going through other tutorials, and most importantly I hope you'll be excited about the possibilities Neo4j can open up in your own projects.

#### What is a graph database?
In the simplest terms, a graph database represents data in terms of a [graph data structure](http://en.wikipedia.org/wiki/Graph_%28abstract_data_type%29), with *nodes* that represent nouns, and *edges* representing connections between those nouns. If you were creating a social network, users, groups, posts, tags, images, reviews, and comments might all be nodes in your database. Edges would connect these nodes in meaningful ways – users connected to their friends, to the posts they write, and to the groups they're members of.

#### What is Neo4j?
Here's the official word: 
>"Neo4j is a highly scalable, robust (fully ACID) native graph database. Neo4j is used in mission-critical apps by thousands of leading startups, enterprises, and governments around the world."

In my experience Neo4j has good documentation and is fairly easy to use in real projects. Working with it tends to be intuitive, and I've actually found it to be pretty fun.

#### Cypher
[Cypher](http://docs.neo4j.org/chunked/stable/cypher-query-lang.html) is the query language used by Neo4j. It's a declarative language that feels a lot like SQL.

Let's jump right in.

This is a node: `()`

Nodes usually have types, called labels. A label might be `:USER` or `:COMMENT`. Nodes can store arbitrary properties in the form of key-value pairs; you might choose to store a user's name, age, or address on his node. Labels and properties are both optional, but in practice you'll probably want at least one of them.

This is an edge: `()-[]->()`. It's sitting between two nodes.

Like nodes, edges have a type, and can store properties: maybe the status of the relationship, or the time the relationship was created. 

To create a node, I instruct my database like this:

    CREATE (:PERSON {name:”Mitch”})
    
I'm saying, "Create a node of type 'Person', storing a 'name' property with the value 'Mitch'." 

If I want to retrieve that node, I do:

    MATCH (me:PERSON {name:”Mitch”})
    RETURN me
    
"Find for me a node, of type PERSON, with a name property with the value 'Mitch'. I will refer to it by the variable 'me'. For now, just show it to me." 

I could also match with less specificity, by leaving out the label, the property, or both.

Creating relatinships works more or less the same way. First, match the two nodes to be connected:

    MATCH (me:PERSON {name:”Mitch”}),
          (jared:PERSON {name:”Jared”})

Then, in the same query, create an edge between them:

    CREATE (me)-[:KNOWS]->(jared)

Now Mitch knows Jared.	

At this point, you know enough to start the [official tutorials](http://docs.neo4j.org/chunked/stable/tutorials-cypher.html) with good context. For the rest of this post, I'm going to focus on the "get excited" part, by showing the power of Neo4j in some (still fairly basic) examples.

#### Example – Social Network
In Mitch's social network, there are many users who follow each other. When a user or a relationship is created, it is cemented in the database as we did above:

    // Create some sample users
    CREATE (mitch:PERSON {name: "Mitch"}),
    (david:PERSON {name: "David"}),          	    
    (jc:PERSON {name: "Joel Cox"}),
    (jared:PERSON {name: "Jared"}),
           ...
    // Create some sample relationships       
    CREATE UNIQUE (mitch)-[:FOLLOWS]->(jared),
    (david)-[:FOLLOWS]->(jared),
    (david)-[:FOLLOWS]->(mitch),
           ...

Note: The whole setup code can be found [on my github](https://github.com/olslash/cypher-queries/blob/master/social.cql). It's excluded here for brevity.

It's simple to find everyone I'm following:

    MATCH (me:PERSON {name:"Mitch"})
    MATCH (me)-[:FOLLOWS]->(other)
    RETURN other
    
Neo4j responds with each node matched as 'other', and any properties on those nodes:
>* {"name": "Jared"}
* {"name": "John"}
* {"name": "Will"}
* {"name": "Park"}

To find everyone who follows me, I just reverse the direction of the edge:

    MATCH (me:PERSON {name:"Mitch"})
    MATCH (me)<-[:FOLLOWS]-(follower)
    RETURN follower, count(DISTINCT follower)

The response, as above, contains the nodes of my followers. I've also asked for a count of unique followers, which Neo4j returns as a simple integer.

Graph databases are especially good at finding "friend of friend" relationships. Here, I tell the database to find friends of my friends (who I do not already follow).

    MATCH (me:PERSON {name:"Mitch"})
    // friendOfFriend is a node two :FOLLOWS relationships away from me.
    MATCH (me)-[:FOLLOWS*2]->(friendOfFriend)
    // I don't want to match anyone who I already follow
    WHERE NOT (me)-[:FOLLOWS]->(friendOfFriend)
    RETURN Distinct friendOfFriend

#### Example – Traveling Salesman
I operate a mobile business. I need to find optimal paths through different cities, based on total distance driven and potential profit.

Here's some setup code (full data on [github](https://github.com/olslash/cypher-queries/blob/master/traveling.cql)) that will create some cities, some roads between them, and my potential profit at each point.

    CREATE (city1:CITY {name: "City1"}),
    (city2:CITY {name: "City2"}),
           ...
    CREATE (city1)-[:ROAD {dist: 20}]->(city3),
    (city1)-[:ROAD {dist: 22}]->(city4),
           ...
    SET city1.value = 100,
	city2.value = 70,
           ... 

To find a path that will generate the most profit with the least driven distance, I can use the following query:

```
// match the starting city
MATCH (start {name:"City1"})

// match possible paths outward
MATCH p=(start)-[:ROAD*0..5]->()

// filter for unique nodes only
WHERE ALL(n in nodes(p) WHERE 1=length(filter(m in nodes(p) WHERE m=n)))

// filter for paths with four cities
AND length(nodes(p)) = 4

RETURN p AS bestPath,

// Sum total distance and total profit on each route
reduce(Distance=0, r in relationships(p) | Distance + r.dist) AS TotalDistance,
reduce(Profit=0, r in nodes(p) | Profit + r.value) AS TotalProfit

// maximize total profit and minimize distance
ORDER BY TotalProfit DESC, TotalDistance ASC
LIMIT 1
```

The database responds that I should drive from City1, to City3, to City6, to City4. I will drive 65 miles and earn 465 dollars, in total.

In the real world you would probably need a much more complicated system, but I hope this contrived example gives some perspective on how powerful basic queries can be.
