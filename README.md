This guide is generally for *nix systems (MacOS and Linux) but should work with a few tweaks for the Windows command line - mostly paths and variable evaluation.

> TODO: Much of the content from this guide should be a docker-compose file, maybe rancher desktop or some mini kubernetes flavor.

Some assumptions are made here: 
* Docker (or podman) is available on the developer's machine
  
* Where things (such as data folders) are - if you want to change these you will need to update some of the steps accordingly.

# 1. Setup Prerequisites

  ## 1.a. Create data folders (Only for initial setup)

    # Create the data folders so that dev data survives restarts
    
    # PostgreSQL
    mkdir -p $HOME/kadena-dev-data/postgres
    # Odoo
    mkdir -p $HOME/kadena-dev-data/odoo

  ## 1.b. Get the dev machine's local ipaddress - this will be used later

    ifconfig

  > TODO: Relying on the ip address to remain constant is a little brittle. This can probably be improved with some sort of local dns server (bind?) and some docker dns parameters.

  ## 1.c. Add www.kadena.local (overwrites any existing entries or creates a new one)

    cat /etc/hosts | grep -v www.kadena.local | sudo tee /etc/hosts; \
    grep -q www.kadena.local /etc/hosts && echo 'nothing to do' || echo '<your_ip_address> www.kadena.local' | sudo tee -a /etc/hosts

  ## 1.d. Add host.local (overwrites the previous setting if it exists)

    cat /etc/hosts | grep -v host.local | sudo tee /etc/hosts; \
    grep -q host.local /etc/hosts && echo 'nothing to do' || echo '<your_ip_address> host.local' | sudo tee -a /etc/hosts


# 2. PostgreSQL
Run postgreSQL in a container (you can also install PostgreSQL on the host and use that instead)

  ## 2.a. Start PostgreSQL
    # Mounts the local data folder into the PostgreSQL container so that data changes survive container (and even host OS) restarts
    # Note: username: user; password: pass
    docker run -d  --name kadena-dev-postgres --rm \
    -v "$HOME"/kadena-dev-data/postgres:/var/lib/postgresql/data -p 5432:5432 \
    -e POSTGRES_USER=user -e POSTGRES_PASSWORD=pass  postgres:15

  ## 2.b. Create required databases (Only for initial setup)
    # Create the Keycloak database
    docker exec -it kadena-dev-postgres sh -c 'PGPASSWORD=pass psql -U user -c "create database keycloak"'

    # Create the Odoo database
    docker exec -it kadena-dev-postgres sh -c 'PGPASSWORD=pass psql -U user -c "create database odoo"'

    # Create the Kadena database
    docker exec -it kadena-dev-postgres sh -c 'PGPASSWORD=pass psql -U user -c "create database kadena"'

# 3. Run Keycloak
  ## 3.a. Start keycloak running as a container

    docker run -d --rm --name kadena-dev-keycloak  -p 8887:8080 \
    -e KEYCLOAK_ADMIN=admin \
    -e KEYCLOAK_ADMIN_PASSWORD=admin \
    -e KC_DB=postgres \
    -e KC_DB_URL=jdbc:postgresql://host.local/keycloak \
    -e KC_DB_USERNAME=user \
    -e KC_DB_PASSWORD=pass \
    -e KC_HOSTNAME=host.local \
    -e KEYCLOAK_BIND_ADDRESS=0.0.0.0 \
    --add-host=host.local:<your_ip_address> \
    quay.io/keycloak/keycloak:25.0.2 start-dev

  ## 3.b. Verify that keycloak is up and running. Visit [http://host.local:8887](http://host.local:8887) in your browser. Login with admin/admin

  ## 3.c. Configure Keycloak (Only for initial setup)

  - Create a realm from the latest kadena-local realm [export file](https://github.com/amihan-net/kadena-policy-enforcer/blob/main/misc/keycloak/kadena-local-realm.json).
    -  'Create Realm' > Resource File (use the export file) > 'Create'
  
  ## 3.d. Generate Client Secret (Only for initial setup)

  - Under the 'kadena-local' realm, click 'Clients' then 'kadena-pep-local'
  - Click the 'Credentials' tab.
  - Click 'Regenerate' beside the 'Client Secrets' text box then click 'Yes' in the resulting pop-up.
  - Copy the regenerated secret and save it somewhere (**this will be used later.**).

# 4. Run Odoo
  > Note: This sets up the open source version. It doesn't include the barcode app out of the box.

  ## 4.a. Initialize odoo database (Only for initial setup)

    # Let the odoo image initialize the database.

    docker run --rm --name kadena-dev-odoo \
    --add-host=host.local:<your_ip_address> \
    -v "$HOME"/kadena-dev-data/odoo:/var/lib/odoo \
    --entrypoint odoo odoo:17 -i base --stop-after-init \
    --db_user user \
    --db_password pass \
    --db_host host.local \
    -d odoo

  ## 4.b. Run Odoo as a container

    # Start Odoo connected to the PostgreSQL database.

    docker run -d -p 8069:8069 --rm --name kadena-dev-odoo \
    -e HOST=host.local \
    -e POSTGRES_DB=odoo \
    -e USER=user \
    -e PASSWORD=pass \
    -v "$HOME"/kadena-dev-data/odoo:/var/lib/odoo \
    --add-host=host.local:<your_ip_address> \
    odoo:17

  ## 4.c. Visit [http://host.local:8069](http://host.local:8069) in your browser. 
  
  - Choose the 'odoo' database
  - Login with admin/admin

  ## 4.d. Activate the required Odoo apps - purchase, inventory, contacts. (Only for initial setup)

  ## 4.e. (Optional) Create a Contact (Only for initial setup or if you need to add users for testing)

  - Menu > Contacts > New
  - Fill up the fields and submit the form
  - Get the id of the created contact

  ## 4.f. Create User(s) in Keycloak (Only for initial setup or if you need to add users for testing)

  - Visit [http://host.local:8887](http://host.local:8887) in your browser. Login with admin/admin
  - Under the 'kadena-local' realm, click 'Users', then 'Create new user'
  - Create the user (select a contact from Odoo and put the id in the Odoo Id field.)
  - Set the user's password in the credentials tab.
  - Add the user's roles in the 'role mapping' tab via the 'Assign Role' button - the Kadena specific roles can be found when you click 'filter realm roles'.

  
  ## 4.e. TODO: Setup OIDC with keycloak
  
 # 5. Run the Backend Services

  ## 5.a. Notification Service

  - Clone the notification service repository [here](https://github.com/amihan-net/notification-service).
  
  > Note: If you want to test sending notifications you may have to change the values in config/notification-service-local.yaml mailgun in the above file.
  
  - Run the app
      
        docker run -d --rm --name kadena-dev-notification-service \
        -p 8080:8080 -v `pwd`:/code -v `pwd`/config/notification-service-local.yaml:/code/.notification-service.yaml -w /code \
        amihanglobal/docker-golang-builder:1.0.0 \
        sh -c 'go run . serve'
  
  ## 5.b. Kadena API Odoo Resources

  - Clone the kadena-api-odoo-resources repository [here](https://github.com/amihan-net/kadena-api-odoo-resources)
  
  - Create a copy .env.example as .env and edit the values as follows (leave the values not listed below as unchanged)
      
        SERVER_JWKS_URL=http://host.local:8887/realms/kadena-local/protocol/openid-connect/certs
        KEYCLOAK_TOKEN_ENDPOINT=/protocol/openid-connect/token
        KEYCLOAK_REALM=kadena-local
        KEYCLOAK_BASE_URL=http://host.local:8887
        CLIENT_ID=kadena-pep-local
        CLIENT_SECRET=<your_client_secret_here>

        # Database configuration values
        DATABASE_HOST=host.local
        DATABASE_USERNAME=user
        DATABASE_PASSWORD=pass
        DATABASE_NAME=kadena

        # Odoo configuration values
        ODOO_AMIN=admin
        ODOO_PASSWORD=admin
        ODOO_DATABASE=odoo
        ODOO_URL=http://host.local:8069/

  - **TODO: Keycloak configuration should no longer be required here since auth is fully delegated to PEP.**
  - Run the app
 
        docker run -d --rm --name kadena-dev-kadena-api-odoo-resources \
        -p 3005:3001 -v `pwd`:/code -w /code \
        --add-host=host.local:<your_ip_address> \
        amihanglobal/docker-golang-builder:1.0.0 \
        sh -c 'go run . serve -e .env'

  ## 5.c. Kadena Orders API
  - Clone the kadena-api-orders repository [here](https://github.com/amihan-net/kadena-api-orders)
  
  - Create a copy .env.example as .env and edit the values as follows
      
        SERVER_JWKS_URL=http://host.local:8887/realms/kadena-local/protocol/openid-connect/certs
        KEYCLOAK_TOKEN_ENDPOINT=/protocol/openid-connect/token
        KEYCLOAK_REALM=kadena-local
        KEYCLOAK_BASE_URL=http://host.local:8887
        CLIENT_ID=kadena-pep-local
        CLIENT_SECRET=<your_client_secret_here>

        # Database configuration values
        DATABASE_HOST=host.local
        DATABASE_USERNAME=user
        DATABASE_PASSWORD=pass
        DATABASE_NAME=kadena

        # Odoo configuration values
        ODOO_SVC_ROOT=http://host.local:3005/api/v1

  - **TODO: Keycloak configuration should no longer be required here since auth is fully delegated to PEP.**
  - Run the app
 
        docker run -d --rm --name kadena-dev-kadena-api-orders \
        -p 3003:3001 -v `pwd`:/code -w /code \
        --add-host=host.local:<your_ip_address> \
        amihanglobal/docker-golang-builder:1.0.0 \
        sh -c 'go run . serve -e .env'

  ## 5.d. Kadena RFQ API
  - Clone the kadena-api-rfq repository [here](https://github.com/amihan-net/kadena-api-rfq)

  - Create a copy .env.example as .env and edit the values as follows
      
        SERVER_JWKS_URL=http://host.local:8887/realms/kadena-local/protocol/openid-connect/certs
        KEYCLOAK_TOKEN_ENDPOINT=/protocol/openid-connect/token
        KEYCLOAK_REALM=kadena-local
        KEYCLOAK_BASE_URL=http://host.local:8887
        CLIENT_ID=kadena-pep-local
        CLIENT_SECRET=<your_client_secret_here>

        # Database configuration values
        DATABASE_HOST=host.local
        DATABASE_USERNAME=user
        DATABASE_PASSWORD=pass
        DATABASE_NAME=kadena

        # Odoo configuration values
        ODOO_SVC_ROOT=http://host.local:3005/api/v1
        NOTIFICATION_SVC_ROOT=http://host.local:8080/api/v1

  - **TODO: Keycloak configuration should no longer be required here since auth is fully delegated to PEP.**

  - Run the app
 
        docker run -d --rm --name kadena-dev-kadena-api-rfq \
        -p 3002:3001 -v `pwd`:/code -w /code \
        --add-host=host.local:<your_ip_address> \
        amihanglobal/docker-golang-builder:1.0.0 \
        sh -c 'go run . serve -e .env'        

## 5.e. Kadena File Upload API
  - Clone the kadena-api-file-upload repository [here](https://github.com/amihan-net/kadena-api-file-upload)

  - Create a copy .env.example as .env and edit the values as follows
      
        SERVER_JWKS_URL=http://host.local:8887/realms/kadena-local/protocol/openid-connect/certs
        KEYCLOAK_TOKEN_ENDPOINT=/protocol/openid-connect/token
        KEYCLOAK_REALM=kadena-local
        KEYCLOAK_BASE_URL=http://host.local:8887
        CLIENT_ID=kadena-pep-local
        CLIENT_SECRET=<your_client_secret_here>

        # Database configuration values
        DATABASE_HOST=host.local
        DATABASE_USERNAME=user
        DATABASE_PASSWORD=pass
        DATABASE_NAME=kadena

        # Odoo configuration values
        ODOO_AMIN=admin
        ODOO_PASSWORD=admin
        ODOO_DATABASE=odoo
        ODOO_URL=http://host.local:8069/

  - **TODO: Keycloak configuration should no longer be required here since auth is fully delegated to PEP.**

  - **TODO: Odoo probably shouldn't be here either.**

  - Run the app
 
        docker run -d --rm --name kadena-dev-kadena-api-file-upload \
        -p 3004:3001 -v `pwd`:/code -w /code \
        --add-host=host.local:<your_ip_address> \
        amihanglobal/docker-golang-builder:1.0.0 \
        sh -c 'go run . serve -e .env'  

# 6. Run the Kadena Workspace

> Note this is a containerized setup but running the kadena workspace application on the host will also work.

- Clone the kadena-workspace repository [here](https://github.com/amihan-net/kadena-workspace)
- Create a file .env and edit as follows
      
      # these are the base urls that the server side fetches will use
      KADENA_API_RFQS_PRIV_BASE_URL=http://host.local:8000/api/rfqs/v1
      KADENA_API_ORDERS_PRIV_BASE_URL=http://host.local:8000/api/orders/v1
      KADENA_API_FILE_UPLOADS_PRIV_BASE_URL=http://host.local:8000/api/file_uploads/v1
      KADENA_API_ODOO_RESOURCES_PRIV_BASE_URL=http://host.local:8000/api/odoo_resources/v1
      # end nodejs only env variables

      NEXT_PUBLIC_KADENA_WORKSPACE_API_BASE_URL=http://www.kadena.local:8000/server

      NEXT_PUBLIC_MAX_FILE_SIZE=3

      # TODO: Update these.
      NEXT_PUBLIC_TABLEAU_URL="https://prod-apnortheast-a.online.tableau.com/#/site/amihanglobalstrategies/views/ForecastingDashboard_TestEnvironment/1YearForecastHorizon"
      NEXT_PUBLIC_TABLEAU_DASHBOARD="https://prod-apnortheast-a.online.tableau.com/#/site/amihanglobalstrategies/views/SupplyPlanningDashboardandSimulator/SupplyPlanningDashboard"
      NEXT_PUBLIC_TABLEAU_PULSE="https://prod-apnortheast-a.online.tableau.com/pulse/site/amihanglobalstrategies"
      # Change this to the link to the inventory module on your local Odoo instance because for some reason this variable is used to link to it. 
      NEXT_PUBLIC_ODOO_URL="http://host.local:8069/web#action=261&model=stock.picking.type&view_type=kanban&cids=1&menu_id=111"
      # Change this to the link to odoo purchase order inside the odoo purchase module
      NEXT_PUBLIC_ODOO_PO_SUBMISSION="http://host.local:8069/web#action=378&model=purchase.order&view_type=list&cids=1&menu_id=236"
      # The below are not supported out of the box in odoo community
      NEXT_PUBLIC_ODOO_QUALITY="https://amihannet-admin-odoo-test-13436936.dev.odoo.com/web#action=426&model=quality.alert.team&view_type=kanban&cids=1&menu_id=259"
      NEXT_PUBLIC_ODOO_BARCODE="https://amihannet-admin-odoo-test-13436936.dev.odoo.com/web#action=434&cids=1&menu_id=276"
      
- Start the application
    
      docker run -d --rm --name kadena-dev-kadena-workspace \
      -p 3001:3001 -v `pwd`:/code -v `pwd`/.env:/code/.env.development \
      -w /code --add-host=www.kadena.local:<your_ip_address> \
      --add-host=host.local:<your_ip_address> \
      node:18-alpine \
      sh -c 'npm i; npm run dev'

# 7. Run the Kadena Policy Enforcer

  - Checkout the kadena-policy-enforcer repository [here](https://github.com/amihan-net/kadena-policy-enforcer)
  
  - Create a copy .env.example as .env and edit the values as follows
       
        SERVER_NAME=www.kadena.local
        KEYCLOAK_BASE_URL=http://host.local:8887
        KEYCLOAK_REALM=kadena-local
        KADENA_CLIENT_ID=kadena-pep-local
        KADENA_CLIENT_SECRET=<your_client_secret>
        KADENA_REDIRECT_URI=http://www.kadena.local:8000/redirect_uri
        KADENA_LOGOUT_PATH=/logout
        KADENA_AFTER_LOGOUT_URI=http://www.kadena.local:8000/
        # please use the internal FQDNS of the services

        KADENA_WORKSPACE_URL=http://host.local:3001
        # for the environmnent variable name, please follow the pattern:
        # KADENA_API_<PLURAL_SERVICE_NAME>_URL ex. KADENA_API_RFQS_URL
        # and the base endpoint should be this pattern: /api/<PLURAL_SERVICE_NAME>/v1/-the-rest-of-the-endpoint-path-
        KADENA_API_RFQS_URL=http://host.local:3002
        KADENA_API_ORDERS_URL=http://host.local:3003
        KADENA_API_FILE_UPLOADS_URL=http://host.local:3004
        KADENA_API_ODOO_RESOURCES_URL=http://host.local:3005
        ODOO_CONTACT_ID_PARAM_NAME=contact-id
        DEBUG=true
      
  - Build an image of the current repo
        
        docker build -t kadena-pep-local -f Dockerfile .

  - Run the image

        docker run -d --rm --name kadena-dev-kadena-policy-enforcer \
        --add-host=host.local:<your_ip_address> \
        -p 8000:80 --env-file=./.env kadena-pep-local

    - Or if you want to be able to reload configuration based on your edits (the details are left as an exercise for the user).
      
          docker run -d --rm --name kadena-dev-kadena-policy-enforcer \
          --add-host=host.local:<your_ip_address> \
          -v `pwd`/nginx/conf/nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf \
          -v `pwd`/nginx/conf/envs.conf:/usr/local/openresty/nginx/conf/envs.conf \
          -v `pwd`/nginx/conf/conf.d:/usr/local/openresty/nginx/conf/conf.d \
          -v `pwd`/nginx/lua:/usr/local/openresty/nginx/lua \
          -p 8000:80 --env-file=./.env kadena-pep-local

# 9. Visit http://www.kadena.local:8000/ in a web browser
