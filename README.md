# Como dockerizar 4 microsserviços.

Olá, pessoal, eu tive dificuldades em estabelecer a conexão do BFF com os demais microsserviços. A solução que achei foi colocar todos os microsserviços dentro da mesma rede. Eu não tenho experiência com Docker, então vou compartilhar a solução que melhor funcionou para mim. Qualquer sugestão ou melhoria, ficarei grato em ouvir.

## Criando uma rede

- Para criar uma nova rede, abra o terminal e digite o comando: **docker network create minha-rede** (neste exemplo, escolhi o nome “minha-rede”, mas fique à vontade para trocar).

## Dockerizando o microsserviço usuario.

- Primeiro criamos um file na raiz do projeto chamado **Dockerfile**, e deve seguir este modelo:

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/[nome do projeto]-0.0.1-SNAPSHOT.jar /app/[nome do projeto].jar

EXPOSE 8080 # 

CMD ["java", "-jar", "/app/[nome do projeto].jar"]
```
**IMPORTANTE: verifique o nome que você escolheu para o projeto. Se você optou pelo gradle, o nome do seu projeto está dentro de “settings.gradle”. Se o projeto está como maven, o nome se encontra no pom.xml na tag “artifactId”.**

**Outra mudança para projetos que utilizam o pom.xml seria o caminho para o jar, em vez de utilizar “build/libs/” utilizamos “COPY target/[nome do projeto]-0.0.1-SNAPSHOT.jar /app/[nome do projeto].jar"**


- O meu projeto se chama “usuario” então meu Dockerfile ficou desta maneira:

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/usuario-0.0.1-SNAPSHOT.jar /app/usuario.jar

EXPOSE 8080

CMD ["java", "-jar", "/app/usuario.jar"]
```
- Agora precisamos criar um file com o nome de **docker-compose.yml** na raiz do projeto.
  
``` yaml
version: '3.8'

services:
  db: # -> criação do banco de dados postgres 
    image: postgres:latest
    container_name: db
    environment:
      POSTGRES_DB: db_projeto
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 1998
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - minha-rede # -> colocando o banco de dados na rede que criamos

  usuario-service: # -> criando o microsserviço que contem nosso jar
    build: .
    container_name: usuario-service # -> nome do microsserviço
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/db_projeto
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: 1998
      SPRING_JPA_HIBERNATE_DDL-AUTO: update
    networks:
      - minha-rede # -> colocando o microsserviço na rede que criamos

volumes:
  postgres_data:

networks:
  minha-rede:
    external: true # -> informando que a rede já existe
```

- Antes de executar qualquer comando Docker, precisamos gerar o jar da aplicação.

- Para aplicações com gradle (IntelliJ).

![image](https://github.com/user-attachments/assets/08d9b543-36e6-4c2a-9274-a29d4df8657b)

- Para aplicações com maven (IntelliJ).

![image](https://github.com/user-attachments/assets/069c9cd5-9aa0-424e-a8d1-be82f0e17800)

- Após gerar o jar, executamos no terminal o comando: **docker-compose up -d --build**

## Dockerizando o microsserviço agendador-tarefas.

- Vamos repetir os passos do microsserviço usuario com algumas mudanças.

- Criamos o **Dockerfile** na raiz do projeto.

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/agendador-tarefas-0.0.1-SNAPSHOT.jar /app/agendador-tarefas.jar

EXPOSE 8081

CMD ["java", "-jar", "/app/agendador-tarefas.jar"]
```

- Criamos o **docker-compose.yml** na raiz do projeto.

``` yaml
version: '3.8'

services:
  mongodb: # -> Criando o banco de dados mongodb
    image: mongo:latest
    container_name: mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: 1998
      MONGO_INITDB_DATABASE: db_agendador_tarefas
    ports:
      - "27017:27017" # -> Porta padrão do mongodb
    volumes:
      - mongo_data:/data/db
    networks:
      - minha-rede  # -> Adicionando o mongodb em nossa rede

  agendador-service: # -> criando o microsserviço que contem nosso jar
    build:
      context: .
      dockerfile: Dockerfile
    container_name: agendador-service # -> nome do microsserviço
    ports:
      - "8081:8081" # -> Porta para subir o agendador-tarefas
    environment:
      SPRING_DATA_MONGODB_URI: mongodb://root:1998@mongodb:27017/db_agendador_tarefas?authSource=admin
    networks:
      - minha-rede # -> Adicionando o microsserviço em nossa rede

volumes:
  mongo_data:

networks:
  minha-rede:
    external: true # -> informando que a rede já existe
```

**IMPORTANTE: Para evitar falhas de autenticação dentro do mongodb, precisamos que a URI do banco de dados esteja correta e com as mesmas credenciais que colocamos em:**

**MONGO_INITDB_ROOT_USERNAME: root**

**MONGO_INITDB_ROOT_PASSWORD: 1998**

**MONGO_INITDB_DATABASE: db_agendador_tarefas**

**URI com os dados corretos: SPRING_DATA_MONGODB_URI: mongodb://root:1998@mongodb:27017/db_agendador_tarefas?authSource=admin**

- Como o "agendador-tarefas" utiliza o Feign Client para validar o token com o microsserviço "usuario", precisamos fazer uma alteração no arquivo application.properties(caso tenha deixado a URL de forma explicita, alterar diretamente no client):
- A nova URL ficou desta maneira: **usuario.url=http://usuario-service:8080**
  - Essa abordagem altera o localhost para o nome do contêiner onde está o microsserviço "usuario".
  - Se tiver dúvidas sobre o nome do contêiner, basta digitar o comando **docker ps -a** no terminal.
    
![image](https://github.com/user-attachments/assets/b3bff4c9-8a2a-4a36-a973-18339f127978)

- Outra alteração, está na URI do mongodb, precisamos alterar de localhost para o nome do contêiner que está rodando o mongodb. Como criamos o contêiner com o nome de "mongodb" a URI ficaria assim: **spring.data.mongodb.uri=mongodb://mongodb:27017/db_agendador_tarefas**

- Uma recomendação para evitar problemas de autenticação com o mongodb, adicionar este comando ao seu application.properties: **spring.data.mongodb.authentication-database=admin**

- Resultado:

![image](https://github.com/user-attachments/assets/bcc0879c-25c6-49d5-aba3-ffe124fdccdb)

- Agora crie o jar e digite o comando: **docker-compose up -d --build**

## Dockerizando o microsserviço notificacao.

- Este microsserviço é mais simples e não faz comunicação direta com nenhum outro microsserviço.

- Criamos o **Dockerfile** na raiz do projeto.

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/notificacao-api-0.0.1-SNAPSHOT.jar /app/notificacao-api.jar

EXPOSE 8082

CMD ["java", "-jar", "/app/notificacao-api.jar"]
```

- Criamos o **docker-compose.yml** na raiz do projeto.

``` yaml
version: '3.8'

services:
  notificacao-service: # -> criando o microsserviço que contem nosso jar
    build: .
    container_name: notificacao-service # -> nome do microsserviço
    ports:
      - "8082:8082" # -> Porta para subir o notificacao-service
    networks:
      - minha-rede # -> Adicionando o microsserviço em nossa rede

networks:
  minha-rede:
    external: true # -> informando que a rede já existe
```

- Após adicionar os arquivos necessários, crie o jar e execute o comando: **docker-compose up -d --build**
  - Se você colocou as configurações do SMTP nas variáveis de ambiente do IntelliJ, após gerar o jar, você precisa colocar as variáveis novamente na versão [clean build].
 
## Dockerizando o microsserviço bff.

- Este microsserviço é bem simples, ele não possui conexão com nenhum banco de dados e se comunica com todos os outros microsserviços.

- Criamos o **Dockerfile** na raiz do projeto.

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY target/bff-0.0.1-SNAPSHOT.jar /app/bff.jar

EXPOSE 8083

CMD ["java", "-jar", "/app/bff.jar"]
```

- Criamos o **docker-compose.yml** na raiz do projeto.

``` yaml
version: '3.8'

services:
  bff-service: # -> criando o microsserviço que contem nosso jar
    build: .
    container_name: bff-service  # -> nome do microsserviço
    ports:
      - "8083:8083" # -> Porta para subir o notificacao-service
    networks:
      - minha-rede # -> Adicionando o microsserviço em nossa rede

networks:
  minha-rede:
    external: true # -> informando que a rede já existe
```

- A alteração mais importante que precisamos fazer é alterar as URLs dos clients que apontam para os microsserviços, usuario,agendador-tarefas e notificacao.
  - Coloque o comando **docker ps -a** para saber o nome exato dos contêineres.

- As URLs modificadas ficaram desta forma:
    - usuario.url=http://usuario-service:8080/usuario
    - tarefa.url=http://agendador-service:8081/tarefas
    - email.url=http://notificacao-service:8082/email
      
![image](https://github.com/user-attachments/assets/04e2558f-0e27-4596-bd18-72790967c363)

- Após adicionar/modificar os arquivos necessários, crie o jar e execute o comando: **docker-compose up -d --build**

# Projeto finalizado

## Os arquivos "Dockerfile" e "docker-compose.yml" estão disponíveis no repositório com o nome de cada microsserviço.

- A estrutura do Docker ficará assim: 

![image](https://github.com/user-attachments/assets/7e3afafc-5273-4b31-b5f5-63b0908b34b5)

- Qualquer dúvida ou sugestão pode me adicionar no discord: mathh1111

