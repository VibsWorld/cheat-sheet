services:
  eventstore:
    image: kurrentplatform/kurrentdb:latest
    environment:
      - KURRENTDB_RUN_PROJECTIONS=All
      - KURRENTDB_INSECURE=true
      - KURRENTDB_ENABLE_ATOM_PUB_OVER_HTTP=true
    ports:
      - "1113:1113"
      - "2113:2113"
    networks:
      - testnet

  trackingeventstore:
    image: kurrentplatform/kurrentdb:latest
    environment:
      - KURRENTDB_RUN_PROJECTIONS=All
      - KURRENTDB_INSECURE=true
      - KURRENTDB_ENABLE_ATOM_PUB_OVER_HTTP=true
    ports:
      - "1114:1113"
      - "2114:2113"
    networks:
      - testnet
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:2113/health/live || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  postgres:
    image: postgres:latest
    volumes:
      - postgres-data:/var/lib/postgresql
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
 
  rabbitmq:
    image: rabbitmq:management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: pvRetailDev
      RABBITMQ_DEFAULT_PASS: pvRetailDev
      RABBITMQ_DEFAULT_VHOST: ParcelVision.Retail
    networks:
      - testnet
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

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
    restart: on-failure

volumes:
  eventstore-data:
    driver: local

  trackingeventstore-data:
    driver: local

  postgres-data:
    driver: local

networks:
  testnet:
    driver: bridge
