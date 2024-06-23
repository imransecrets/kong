# kong

https://www.youtube.com/watch?v=Q5_hfGY672U&list=PLplW4d4HPsEINijsTm9ZONDcEdtSq5Hg9&index=2
git link for documents
https://gist.github.com/pantsel/73d949774bd8e917bfd3d9745d71febf

Take code from above repository and little change

    version: "3"
    
    networks:
     kong-net:
      driver: bridge
    
    services:
    
      #######################################
      # Postgres: The database used by Kong (only change network according to my codes)
      #######################################
      
      kong-database:
        image: postgres:9.6
        restart: always
        networks:
          default
            driver: bridge
        environment:
          POSTGRES_USER: kong
          POSTGRES_DB: kong
        ports:
          - "5432:5432"
        healthcheck:
          test: ["CMD", "pg_isready", "-U", "kong"]
          interval: 5s
          timeout: 5s
          retries: 5
    
      #######################################
      # Kong database migration    (remove network from below code so comment out or delete)
      #######################################
      kong-migration:
        image: kong:latest
        command: "kong migrations bootstrap"
       # networks:
       #   - kong-net
        restart: on-failure
        environment:
          KONG_PG_HOST: kong-database
        links:
          - kong-database
        depends_on:
          - kong-database
    
      #######################################
      # Kong: The API Gateway   (remove network from below code so comment out or delete)
      #######################################
      kong:
        image: kong:latest
        restart: always
        # networks:
         #  - kong-net
        environment:
          KONG_PG_HOST: kong-database
          KONG_PROXY_LISTEN: 0.0.0.0:8000
          KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
          KONG_ADMIN_LISTEN: 0.0.0.0:8001
        depends_on:
          - kong-migration
          - kong-database
        healthcheck:
          test: ["CMD", "curl", "-f", "http://kong:8001"]
          interval: 5s
          timeout: 2s
          retries: 15
        ports:
          - "8001:8001"
          - "8000:8000"
    
      #######################################
      # Konga database prepare
      #######################################
      konga-prepare:
        image: pantsel/konga:next
        command: "-c prepare -a postgres -u postgresql://kong@kong-database:5432/konga_db"
        networks:
          - kong-net
        restart: on-failure
        links:
          - kong-database
        depends_on:
          - kong-database
    
      #######################################
      # Konga: Kong GUI
      #######################################
      konga:
        image: pantsel/konga:next
        restart: always
        networks:
            - kong-net
        environment:
          DB_ADAPTER: postgres
          DB_HOST: kong-database
          DB_USER: kong
          TOKEN_SECRET: km1GUr4RkcQD7DewhJPNXrCuZwcKmqjb
          DB_DATABASE: konga_db
          NODE_ENV: production
        depends_on:
          - kong-database
        ports:
          - "1337:1337"
