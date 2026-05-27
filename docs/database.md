# How to work with databases

## View Database locally

**Install pgAdmin4:**

```
sudo rpm -i https://ftp.postgresql.org/pub/pgadmin/pgadmin4/yum/pgadmin4-fedora-repo-2-1.noarch.rpm
sudo dnf install pgadmin4-desktop
```

_You can also use snap-packet_

**Port forward the database to your local machine:**

```
kubectl port-forward -n cnpg-database svc/postgres-rw 5432:5432
```

**Register database in pgAdmin4:**

- click register new server
- give a name
- connection tab input: `localhost` with your `postgres` password

## Add database

**Add user to database cluster:**

1. Inside `cnpg-database` add a new role
   ```yaml
   managed:
     roles:
       - name: authentik
         ensure: present
         login: true
         passwordSecret:
           name: authentik-db-secret
   ```
2. Add role **Sealed** credentials to `app-db-secrets.yaml`
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: <ROLE>-db-credentials
     namespace: cnpg-database
   type: kubernetes.io/basic-auth
   stringData:
     username: <ROLE_NAME>
     password: <STRONG_PASSWORD>
   ```
3. Copy the **Secret** (_not_ the SealedSecret) to the app service and use it there
