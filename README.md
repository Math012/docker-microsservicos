# Como dockerizar 4 microsserviços.

Olá pessoal, eu tive dificuldades em estabelecer a conexão do BFF com os demais microsserviços. A solução que eu achei foi colocar todos os microsserviços dentro da mesma rede.
Eu não tenho experiencia com docker, então vou compartilhar a solução que melhor funcionou para mim. Qualquer sugestão ou melhoria ficarei grato em ouvir.

## Criando uma rede

- Para criar uma nova rede, abra o terminal e digite o comando: **docker network create minha-rede** (neste exemplo eu escolhi o nome "minha-rede" mas fique a vontade para trocar)

## Dockerizando o microsserviço usuario.

- Primeiro criamos um file na raiz do projeto chamado **Dockerfile**, e deve seguir este modelo:

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/[nome do projeto]-0.0.1-SNAPSHOT.jar /app/[nome do projeto].jar

EXPOSE 8080 # 

CMD ["java", "-jar", "/app/[nome do projeto].jar"]
```
**IMPORTANTE: verifique o nome que você escolheu para o projeto, se voce optou pelo gradle, o nome do seu projeto está dentro de "settings.gradle" se o projeto esta como maven, o nome se encontra no pom.xml dentro da tag "artifactId".**

**Outra mudança para projetos que utilizam o pom.xml seria o caminho para o jar, em vez de utilizar "build/libs/" utilizamos "COPY target/[nome do projeto]-0.0.1-SNAPSHOT.jar /app/[nome do projeto].jar"**


- O meu projeto se chama "usuario" então meu dockerfile ficou desta maneira:

``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/usuario-0.0.1-SNAPSHOT.jar /app/usuario.jar

EXPOSE 8080

CMD ["java", "-jar", "/app/usuario.jar"]
```
- Agora precisamos criar um file com o nome de **docker-compose.yml** dentro da raiz do projeto
  
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
      - minha-rede # -> colocando o banco de dados dentro da rede que criamos

  usuario-service: # -> criando o microsserviço que contem nosso jar
    build: .
    container_name: usuario-service
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/db_projeto
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: 1998
      SPRING_JPA_HIBERNATE_DDL-AUTO: update
    networks:
      - minha-rede # -> colocando o microsserviço de dados dentro da rede que criamos

volumes:
  postgres_data:

networks:
  minha-rede:
    external: true # -> informando que a rede já existe
```

- Antes de executar qualquer comando docker, precisamos gerar o jar da aplicação

- Para aplicações com gradle (intellij)

![image](https://github.com/user-attachments/assets/08d9b543-36e6-4c2a-9274-a29d4df8657b)

- Para aplicações com maven (intellij)

![image](https://github.com/user-attachments/assets/069c9cd5-9aa0-424e-a8d1-be82f0e17800)

- Após gerar o jar, executamos no terminal o comando: docker-compose up -d --build

## Dockerizando o microsserviço agendador-tarefas.

- Vamos repetir os passos do microsserviço usuario com algumas mudanças.

- Criamos o **Dockerfile** dentro da raiz do projeto.


``` yaml
FROM openjdk:17-jdk-alpine

WORKDIR /app

COPY build/libs/agendador-tarefas-0.0.1-SNAPSHOT.jar /app/agendador-tarefas.jar

EXPOSE 8081

CMD ["java", "-jar", "/app/agendador-tarefas.jar"]
```

- criamos o **docker-compose.yml** dentro da raiz do projeto.

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
      - minha-rede  # -> Adicionando o microsserviço em nossa rede

  agendador-service:
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
