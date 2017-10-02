# Pixels Camp Demo Scratch Pad

## Create Docker Containers
```
docker run -d -e POSTGRES_PASSWORD=changeit --name postgres postgres:9.6-alpine
```
```
docker run --link postgres -p 8200:8200 -e VAULT_DEV_ROOT_TOKEN_ID=123456 vault
```

## Execute CLI
```
docker exec -it <VAULT_CONTAINER_ID> /bin/sh
```

## Set Vault Server Address
```
export VAULT_ADDR='http://0.0.0.0:8200'
export VAULT_TOKEN=123456
```

## Part 1 (Simple Secret)

### Mount Databases plugin
```
vault mount database
```

### Mount at a different point (use this for multiple databases)
```
vault mount -path database2 database
```

### Configure a readonly role
```
vault write database/roles/readonly \
   db_name=postgresql \
   creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
   default_ttl="1h" \
   max_ttl="24h"
```
### Configure Postgres Connnection
```
vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="readonly" \
    connection_url="postgresql://postgres:changeit@postgres:5432/?sslmode=disable"
```

### Get Credentials to access Database
```
vault read database/creds/readonly
```

```
docker exec -it <POSTGRES_CONTAINER_ID> psql -U <user> -W <password> -d postgres
```

```
SELECT * FROM pg_users;
```

## Part 2 (Policies)

### Create a new policy for Alice
```
curl -X PUT --header "X-Vault-Token: 123456" --data @policy.json http://localhost:8200/v1/sys/policy/alice-policy
```

### Test-out with default policy (FAILS)
```
vault token-create -policy=default
```

#### Open a new Terminal (DO NOT SET VAULT_TOKEN Variable)

```
vault auth <TOKEN>
```

```
vault read secret/foo
```
### Test-out with created policy (SUCCEEDS)
```
vault token-create -policy=alice-policy
```
#### Open a new Terminal (DO NOT SET VAULT_TOKEN Variable)
```
vault auth <TOKEN>
vault read secret/foo
```

### Enable AppRoles
```
vault auth-enable approle
```

### Create a Role
```
vault write auth/approle/role/alicerole secret_id_ttl=10m token_num_uses=10 token_ttl=20m token_max_ttl=30m secret_id_num_uses=2 policies=alice-policy
```

### Read Secret ID from that role
```
vault read auth/approle/role/alicerole/role-id
```

### Generate a Secret ID (Each one can only be used twice to get a Token)
```
vault write -f auth/approle/role/alicerole/secret-id
```

### Login with AppRole/Generate a Token (Repeat 3 times - 3rd time should fail)
```
vault write auth/approle/login role_id=<role_id> secret_id=<secret_id>
```

### Authenticate using token
```
vault auth <token> 
```

### Check if can read secret (Should Succeed)
```
vault read secret/foo
```

