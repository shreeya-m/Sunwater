# SUMMARY
Created a shortest path algorithm using Neo4j and python that displayed the shortest path between two staffers that worked in different offices, while also creating an alternative path

# Neo4j Documentation
Began with importing data into Neo4j and creating nodes for each staffer which included properties such as Office, Name, Id, and JobTitle. Created two separate relationships that showed when a staffer, based on the Name and ID, changed jobs or offices (SWITCHED_JOB_OFFICE). The second relationship showed whether a staffer worked in the same office as another staffer (colleague). I also indexed the first name, offices and dates in order to make the queries faster. 

Full list of properties key:
- enddate
- first_name
- id
- job_title
- office_name
- start_date
- weight

# example of query used to import csv into neo4j
CALL apoc.periodic.iterate(
  'CALL apoc.load.csv("file:///legis1first.csv", {headers: true}) YIELD map AS row RETURN row',
  'MERGE (s:staffer {id: row.person_id})
   SET s.name = row.name, 
       s.startdate = date(row.start_date),
       s.enddate = date(row.end_date)',
  {batchSize: 1000, parallel: true}
) YIELD total
RETURN total

CREATE INDEX ON :staffer(office);
CREATE INDEX ON :staffer(name);

# Creating relationships
In order to maximize storage, I used apox.iterate to iterate and create relationships in smaller batch sizes (1000) as I faced problems with loading the data and the time it would take. 

CALL apoc.periodic.iterate(
    "MATCH (s1:staffer)
     RETURN s1",
    "MATCH (s2:staffer)
     WHERE s1.office = s2.office AND id(s1) < id(s2)
     MERGE (s1)-[r:colleague]->(s2)",
    {batchSize:1000, parallel:true}
)

Similarly, I used apoc to delete duplicated relationships. 

CALL apoc.periodic.iterate(
  "MATCH (s1:staffer)-[r:colleague]->(s2:staffer)
   WHERE id(s1) < id(s2)
   RETURN s1, s2, collect(r) as rels
   ORDER BY id(s1), id(s2)",
  
  "FOREACH (rel IN tail(rels) | DELETE rel)",
  
  { batchSize: 1000, parallel: true }
)

# Creating weights
A major focus of this project was creating adequate weights based on the number of days two staffers worked together, showing how much more likely they would be to work with one another. Therefore, I create a weight node which would display a number based on this overlap.

CALL apoc.periodic.iterate(
  'MATCH (s1:staffer), (s2:staffer)
   WHERE id(s1) < id(s2) AND s1.office = s2.office
   RETURN s1, s2',
  'WITH s1, s2,
       date(s1.startdate) AS s1_start,
       date(s1.enddate) AS s1_end,
       date(s2.startdate) AS s2_start,
       date(s2.enddate) AS s2_end,
       CASE WHEN date(s1.startdate) > date(s2.startdate) THEN date(s1.startdate) ELSE date(s2.startdate) END AS overlap_start,
       CASE WHEN date(s1.enddate) < date(s2.enddate) THEN date(s1.enddate) ELSE date(s2.enddate) END AS overlap_end
   WITH s1, s2,
        duration.inDays(overlap_start, overlap_end).days AS overlap_days
   WHERE overlap_days > 0
   MERGE (s1)-[r:colleague]->(s2)
   SET r.weight = overlap_days / 365.0',
  {batchSize: 50, parallel: true}
);


# Shortest path algorithm in Neo4j
After creating the weights, I used Neo4j's built in shortest path function to display how the algorithm with show the shortest path between two different staffers in different offices.

WITH 'office name' AS officeId1, 'office name 2' AS officeId2
// Match all staffers from the first office
MATCH (staffer1:staffer {office_name: officeId1})
// Match all staffers from the second office
MATCH (staffer2:staffer {office_name: officeId2})
// Find the shortest path between staffers in the two offices
MATCH path = shortestPath((staffer1)-[:colleague|SWITCHED_JOB_OFFICE*]-(staffer2))
// Return the shortest path
RETURN path
ORDER BY length(path) ASC
LIMIT 1

# Streamlit and Python
Initially, I tried to use Plotly to display this algorithm with user input, but switched to Python when this was unsuccessful. Streamlit helped to create an easily accessible user interface, while python helped me import the data from Neo4j, and display as needed based on the user input. This code is displayed in the python file.
