# aws-iam-neo4j

## Clone this repo

```
git clone https://github.com/gabemarshall/aws-iam-neo4j.git
cd aws-iam-neo4j/docker
```

## Export IAM Settings of you accout

Run the following command to extract all your AWS IAM settings:

```
aws iam get-account-authorization-details > ../auth.json
```


## Start Docker neo4j with apoc plugin

```
docker run -d \
    -p 7474:7474 -p 7687:7687 \
    -v $PWD/data:/var/lib/neo4j/data -v $PWD/plugins:/var/lib/neo4j/plugins \
    --name neo4j-apoc \
    -e NEO4J_apoc_export_file_enabled=true \
    -e NEO4J_apoc_import_file_enabled=true \
    -e NEO4J_apoc_import_file_use__neo4j__config=true \
    neo4j
```

## Once neo4j is ready, login to neo4j and change your password (default is neo4j:neo4j)

```
Browse to http://localhost:7474/
```

## Copy your auth.json and aws_iam_load.query files into the running container

```
docker cp ../aws_iam_load.cypher neo4j-apoc:/var/lib/neo4j/aws_iam_load.query
docker cp ../auth.json neo4j-apoc:/var/lib/neo4j/import/auth.json
```

## Run the cypher query

```
docker exec -it neo4j-apoc /bin/bash
cat aws_iam_load.query | cypher-shell -u neo4j -p [your password]

# If the above fails, load cypher-shell and manually paste in the contents of aws_iam_load.query
```

## Graph Schema

![AWS IAM Schema](./db_schema.png)


## Relevant Cypher Queries

### Show me all Users
```
MATCH (u:IAM_User) RETURN u
```

### Show me all Users and their Groups
```
MATCH (u:IAM_User)-[]->(g:IAM_Group) RETURN u,g
```

### Show me all Groups with at least one User
```
MATCH (u:IAM_User)-[:MEMBER_OF]->(g:IAM_Group) 
WITH count(u) as n,u,g
WHERE n > 0
RETURN u,g
```

### Show me a specific Policy and all related Relationships 
```
MATCH (p:IAM_Policy)-[]->(n)
WHERE p.name = '<PolicyName>'
RETURN n,p 
```


### Show me all Roles
```
MATCH (r:IAM_Role) RETURN r
```

### Show me all Roles with at least one Policy attached
```
MATCH (r:IAM_Role)-[]->(p:IAM_Policy) 
WITH count(p) as n, r, p
WHERE n > 0 
RETURN r,p
```

### Show me the Policies, its Resources and Actions, and Users with Access
```
MATCH (r:IAM_Policy_Resource)<-[:HAS_RESOURCE]-(p:IAM_Policy)<-[:HAS_POLICY]-(u:IAM_User)
MATCH (a:IAM_Policy_Action)<-[:HAS_ACTION]-(p:IAM_Policy)
RETURN r,p,u,a
```
![](./policy_users_actions_resources.png)