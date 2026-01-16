# Analisando Dados de Rede Social com Base em Consultas de Grafos

 - Essa modelagem de dados em grafos, foi desenvolvida no <b>Neo4j</b>.<br>
 - Abaixo está o script Cypher: <br><br>

// SCRIPT DE IMPORTÇÃO DE REDE SOCIAL A PARTIR DE ARQUIVOS<br>
// 1. CONFIGURAÇÃO INICIAL<br>

CREATE CONSTRAINT IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;<br>
CREATE CONSTRAINT IF NOT EXISTS FOR (p:Post) REQUIRE p.id IS UNIQUE;<br>
CREATE CONSTRAINT IF NOT EXISTS FOR (t:Topic) REQUIRE t.name IS UNIQUE;<br>
CREATE CONSTRAINT IF NOT EXISTS FOR (h:Hashtag) REQUIRE h.tag IS UNIQUE;<br><br>

CREATE INDEX IF NOT EXISTS FOR (u:User) ON (u.username);<br>
CREATE INDEX IF NOT EXISTS FOR (p:Post) ON (p.created_at);<br>
CREATE INDEX IF NOT EXISTS FOR (p:Post) ON (p.score);<br><br>

// 2. LIMPAR DADOS EXISTENTES (OPCIONAL - APENAS PARA RESET)<br>
// MATCH (n) DETACH DELETE n;<br>

// 3. IMPORTAR DADOS DE USUÁRIOS<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/users.csv' AS row<br>
CREATE (u:User {
    id: row.id,
    username: row.username,
    name: row.name,
    age: toInteger(row.age),
    city: row.city,
    registration_date: date(row.registration_date),
    interests: split(row.interests, '|')
});<br>


// 4. IMPORTAR RELAÇÕES DE SEGUIDORES<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/follows.csv' AS row<br>
MATCH (follower:User {id: row.follower_id})
MATCH (followed:User {id: row.followed_id})
CREATE (follower)-[:FOLLOWS {
    since: date(row.since),
    strength: toFloat(row.strength)
}]->(followed);<br>


// 5. IMPORTAR POSTS<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/posts.csv' AS row<br>
MATCH (author:User {id: row.user_id})
CREATE (p:Post {
    id: row.id,
    content: row.content,
    created_at: datetime(row.created_at),
    score: toFloat(row.score),
    sentiment: row.sentiment,
    likes_count: 0,
    shares_count: 0
})<br>
CREATE (author)-[:POSTED {timestamp: datetime(row.created_at)}]->(p);<br>


// 6. IMPORTAR E PROCESSAR HASHTAGS DOS POSTS<br>
// Primeiro, extrair hashtags únicas dos posts<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/posts.csv' AS row<br>
WITH row.id as post_id, split(row.hashtags, ',') as hashtag_list<br>
UNWIND hashtag_list AS hashtag<br>
WITH post_id, trim(hashtag) as clean_hashtag<br>
WHERE clean_hashtag <> ''<br>
MERGE (h:Hashtag {tag: clean_hashtag})<br>
ON CREATE SET h.usage_count = 1<br>
ON MATCH SET h.usage_count = h.usage_count + 1<br>
WITH h, post_id<br>
MATCH (p:Post {id: post_id})<br>
MERGE (p)-[:TAGGED_WITH]->(h);<br>


// 7. IMPORTAR INTERESSES/TOPICOS DOS USUÁRIOS<br>
// Primeiro, criar tópicos únicos baseados nos interesses dos usuários<br>
MATCH (u:User)<br>
UNWIND u.interests AS interest<br>
WITH DISTINCT trim(interest) as topic_name<br>
WHERE topic_name <> ''<br>
MERGE (t:Topic {name: topic_name});<br>

// Criar relações de interesse<br>
MATCH (u:User)<br>
UNWIND u.interests AS interest<br>
WITH u, trim(interest) as topic_name<br>
WHERE topic_name <> ''<br>
MATCH (t:Topic {name: topic_name})<br>
MERGE (u)-[:INTERESTED_IN {level: 0.7 + rand() * 0.3}]->(t);<br>


// 8. IMPORTAR INTERAÇÕES (LIKES E COMPARTILHAMENTOS)<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/interactions.csv' AS row<br>
WITH row<br>
WHERE row.interaction_type = 'LIKE'<br>
MATCH (u:User {id: row.user_id})<br>
MATCH (p:Post {id: row.post_id})<br>
CREATE (u)-[:LIKES {timestamp: datetime(row.timestamp)}]->(p);<br>

// Importar SHARES<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/interactions.csv' AS row<br>
WITH row<br>
WHERE row.interaction_type = 'SHARE'<br>
MATCH (u:User {id: row.user_id})<br>
MATCH (p:Post {id: row.post_id})<br>
CREATE (u)-[:SHARED {timestamp: datetime(row.timestamp), platform: row.platform}]->(p);<br>

// Importar COMMENTS<br>
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/walmirpacheco/Consulta-de-Grafos/main/import/interactions.csv' AS row<br>
WITH row<br>
WHERE row.interaction_type = 'COMMENT'<br>
MATCH (u:User {id: row.user_id})<br>
MATCH (p:Post {id: row.post_id})<br>
CREATE (c:Comment {id: randomUUID(), content: row.content, timestamp: datetime(row.timestamp), sentiment: row.sentiment})<br>
CREATE (u)-[:COMMENTED_ON]->(c)-[:ON_POST]->(p);<br>


// 9. ATUALIZAR CONTADORES NOS POSTS<br>
// Atualizar contadores de likes<br>
MATCH (p:Post)<br>
SET p.likes_count = COUNT { (p)<-[:LIKES]-() };<br>

// Atualizar contadores de shares<br>
MATCH (p:Post)<br>
SET p.shares_count = COUNT { (p)<-[:SHARED]-() };<br>


// 10. VERIFICAR DADOS IMPORTADOS<br>
// Estatísticas básicas<br>
MATCH (u:User) RETURN count(u) as TotalUsuarios;<br>
MATCH (p:Post) RETURN count(p) as TotalPosts;<br>
MATCH ()-[f:FOLLOWS]->() RETURN count(f) as TotalRelacoesFollow;<br>
MATCH ()-[l:LIKES]->() RETURN count(l) as TotalLikes;<br>
MATCH ()-[s:SHARED]->() RETURN count(s) as TotalShares;<br>

### O resultado do grafo geral é essa imagem abaixo:<br>

![Vizualizacao Geral][img1]

### O resultado do grafo mostrando quem relaciona com quem na imagem abaixo:<br>
[img1]: images/Visualizacao_Geral.png
![Vizualizacao Quem Segue Quem][img2]


### O resultado do grafo mostrando de como chegar entre o Lucas Almeida e a Beatriz Alves, e seus amigos mais próximos, veja a imagem abaixo:<br>
[img2]: images/Visualizacao_quem_segue_quem.png


![Vizualizacao Lucas Beatriz][img3]

[img3]: images/Visualizacao_Lucas_Beatriz.png