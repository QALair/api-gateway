# api-gateway

couldnt find a place to put the docker compose file, so I will just put it here, not going to put it on git CI right now, gotta study another things

version: '3.4'

services:
  zipkin-server:
    image: openzipkin/zipkin:2.23.2
    ports:
      - 9411:9411
    restart: always
    depends_on:
      - rabbit-mq
    environment:
      RABBIT_URI: amqp://guest:guest@rabbit-mq:5672
    networks:
      - joao-network
      
  rabbit-mq:
    image: rabbitmq:3.8.14-management
    ports:
      - 5672:5672
      - 15672:15672
    networks:
      - joao-network
      
  cambio-db:
    image: mysql:8.0
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    environment:
      TZ: America/Sao_Paulo
      MYSQL_ROOT_PASSWORD: somepassword
      MYSQL_USER: docker
      MYSQL_PASSWORD: somepassword
      MYSQL_DATABASE: cambio_service
      MYSQL_ROOT_HOST: '%'
      MYSQL_TCP_PORT: 3308
    ports:
      - 3308:3308
    expose:
      - 3308
    networks:
      - joao-network
      
  book-db:
    image: mysql:8.0
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    environment:
      TZ: America/Sao_Paulo
      MYSQL_ROOT_PASSWORD: somepassword
      MYSQL_USER: docker
      MYSQL_PASSWORD: somepassword
      MYSQL_DATABASE: book_service
      MYSQL_ROOT_HOST: '%'
      MYSQL_TCP_PORT: 3310
    ports:
      - 3310:3310
    expose:
      - 3310
    networks:
      - joao-network

  naming-server:
    image: joaotn/naming-server:0.0.1-SNAPSHOT
    ports:
      - 8761:8761
    networks:
      - joao-network

  api-gateway:
    image: joaotn/api-gateway:0.0.1-SNAPSHOT
    ports:
      - 8765:8765
    depends_on:
      - naming-server
      - rabbit-mq
    environment:
      EUREKA.CLIENT.SERVICEURL.DEFAULTZONE: http://naming-server:8761/eureka
      SPRING.ZIPKIN.BASEURL: http://zipkin-server:9411/
      RABBIT_URI: amqp://guest:guest@rabbit-mq:5672
      SPRING_RABBITMQ_HOST: rabbit-mq
      SPRING_ZIPKIN_SENDER_TYPE: rabbit      
    networks:
      - joao-network
      
  cambio-service:
    image: joaotn/cambio-service
    restart: always
    build:
      context: .
      dockerfile: cambio-service/Dockerfile
    environment:
      TZ: America/Sao_Paulo
      EUREKA.CLIENT.SERVICEURL.DEFAULTZONE: http://naming-server:8761/eureka
      SPRING.ZIPKIN.BASEURL: http://zipkin-server:9411/
      SPRING.DATASOURCE.URL: jdbc:mysql://cambio-db:3308/cambio_service?useSSL=false&serverTimezone=UTC&enabledTLSProtocols=TLSv1.2
      SPRING.DATASOURCE.USERNAME: root
      SPRING.DATASOURCE.PASSWORD: somepassword
      SPRING.FLYWAY.URL: jdbc:mysql://cambio-db:3308/cambio_service?useSSL=false&serverTimezone=UTC&enabledTLSProtocols=TLSv1.2
      SPRING.FLYWAY.USER: root
      SPRING.FLYWAY.PASSWORD: somepassword
      RABBIT_URI: amqp://guest:guest@rabbit-mq:5672
      SPRING_RABBITMQ_HOST: rabbit-mq
      SPRING_ZIPKIN_SENDER_TYPE: rabbit        
    ports:
      - 8000:8000
    depends_on:
      - naming-server
      - cambio-db
      - rabbit-mq      
    networks:
      - joao-network
      
  book-service:
    image: joaotn/book-service
    restart: always
    build:
      context: .
      dockerfile: book-service/Dockerfile
    environment:
      TZ: America/Sao_Paulo
      EUREKA.CLIENT.SERVICEURL.DEFAULTZONE: http://naming-server:8761/eureka
      SPRING.ZIPKIN.BASEURL: http://zipkin-server:9411/
      SPRING.DATASOURCE.URL: jdbc:mysql://book-db:3310/book_service?useSSL=false&serverTimezone=UTC&enabledTLSProtocols=TLSv1.2
      SPRING.DATASOURCE.USERNAME: root
      SPRING.DATASOURCE.PASSWORD: somepassword
      SPRING.FLYWAY.URL: jdbc:mysql://book-db:3310/book_service?useSSL=false&serverTimezone=UTC&enabledTLSProtocols=TLSv1.2
      SPRING.FLYWAY.USER: root
      SPRING.FLYWAY.PASSWORD: somepassword
      RABBIT_URI: amqp://guest:guest@rabbit-mq:5672
      SPRING_RABBITMQ_HOST: rabbit-mq
      SPRING_ZIPKIN_SENDER_TYPE: rabbit        
    ports:
      - 8100:8100
    depends_on:
      - naming-server
      - book-db
      - rabbit-mq      
    networks:
      - joao-network
networks:
  joao-network:
    driver: bridge
