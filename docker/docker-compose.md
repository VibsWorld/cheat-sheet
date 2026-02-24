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

More Health Checks Examples
```
services:
  eventstore:
    image: kurrentplatform/kurrentdb:latest
    environment:
      - KURRENTDB_RUN_PROJECTIONS=All
      - KURRENTDB_INSECURE=true
      - KURRENTDB_ENABLE_ATOM_PUB_OVER_HTTP=true
      - KURRENTDB_LOG_LEVEL=Error
    ports:
      - "1113:1113"
      - "2113:2113"
    networks:
      - testnet
    restart: on-failure

  trackingeventstore:
    image: kurrentplatform/kurrentdb:latest
    environment:
      - KURRENTDB_RUN_PROJECTIONS=All
      - KURRENTDB_INSECURE=true
      - KURRENTDB_ENABLE_ATOM_PUB_OVER_HTTP=true
      - KURRENTDB_LOG_LEVEL=Error
    ports:
      - "1114:1113"
      - "2114:2113"
    networks:
      - testnet
    restart: on-failure

  postgres:
    image: postgres:latest
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=changeit
      - POSTGRES_DB=postgres
    ports:
      - "5432:5432"
    networks:
      - testnet
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    restart: on-failure

  rabbitmq:
    image: rabbitmq:4-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
      RABBITMQ_DEFAULT_VHOST: guest
    networks:
      - testnet
    # Enable the plugin offline, then start the server
    command: bash -c "rabbitmq-plugins enable --offline rabbitmq_consistent_hash_exchange rabbitmq_management && rabbitmq-server"
    healthcheck:
      test: ["CMD-SHELL", "rabbitmq-diagnostics -q ping"]
      interval: 10s
      timeout: 10s
      retries: 12
      start_period: 60s
    restart: on-failure

  unleash:
    image: unleashorg/unleash-server:latest
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - DATABASE_URL=postgres://postgres:changeit@postgres:5432/postgres
      - DATABASE_SSL=false
      - INIT_FRONTEND_API_TOKENS=default:development-unleash-insecure-api-token
      - SECRET_NAMES=default
    ports:
      - "4242:4242"
    networks:
      - testnet
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://localhost:4242/health"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    restart: on-failure

  pando-api:
    build:
      context: ../../src/Pando.Host
      dockerfile: pando.Dockerfile
    depends_on:
      eventstore:
        condition: service_started
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      unleash:
        condition: service_healthy
    environment:
      - ASPNETCORE_ENVIRONMENT=docker
      - ASPNETCORE_URLS=http://*:8111
    ports:
      - "8111:8111"
    networks:
      - testnet
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8111/swagger/index.html"]
      interval: 10s
      timeout: 10s
      retries: 12
      start_period: 60s
    restart: on-failure

networks:
  testnet:
    driver: bridge
```
