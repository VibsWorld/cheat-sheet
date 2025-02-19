Healthchecks for Postgres, Rabbitmq and normal image
````
services:
  postgres:
    image: postgres
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=changeit
      - POSTGRES_DB=postgres
    ports:
      - "5432:5432"
    networks:
      - testnet
    healthcheck:
      test:
        [
          "CMD",
          "pg_isready",
          "--username=postgres",
          "--host=127.0.0.1",
          "--port=5432"
        ]
      interval: 2s
      timeout: 1m
      retries: 5
      start_period: 10s

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      - RABBITMQ_DEFAULT_USER=pvRetailDev
      - RABBITMQ_DEFAULT_PASS=pvRetailDev
      - RABBITMQ_DEFAULT_VHOST=ParcelVision.Retail
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - testnet
    healthcheck:
      test: rabbitmq-diagnostics check_port_connectivity
      interval: 30s
      timeout: 30s
      retries: 10 

  unleash:
    image: unleashorg/unleash-server:6.5.3
    restart: on-failure
    depends_on:
      postgres:
        condition: service_healthy
    command: [ "node", "index.js" ]
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:4242/health || exit 1
      interval: 1s
      timeout: 1m
      retries: 5
      start_period: 15s
    env_file:
      - service.env
    ports:
      - 4242:4242
    networks:
      - testnet
      
  laurel:
    depends_on:
       rabbitmq:
          condition: service_healthy
    restart: on-failure
    links: 
        - postgres
        - rabbitmq
    build:
      context: ../Laurel.Host
      dockerfile: laurel.Dockerfile
    environment:
      - ASPNETCORE_ENVIRONMENT=docker
      - ASPNETCORE_URLS=http://*:7001
    ports:
      - "7001:7001"
    networks:
      - testnet
      
networks:
  testnet:
    driver: bridge
````
