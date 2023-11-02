# Setup HashiCorp Vault on docker

Vault secures, stores, and tightly controls access to tokens, passwords, certificates, API keys, and other secrets in modern computing. Vault is primarily used in production environments to manage secrets. Vault is a complex system that has many different pieces. There is a clear separation of components that are inside or outside of the security barrier. Only the storage backend and the HTTP API are outside, all other components are inside the barrier.

![Vault_architecture](https://user-images.githubusercontent.com/17578897/171566789-ba0a69ae-4a4b-4cfa-a4fd-21d997c0d818.png)

Figure 1: Architecture of Vault and Spring App (Click to enlarge)

The storage backend is untrusted and is used to durably store encrypted data. When the Vault server is started, it must be provided with a storage backend so that data is available across restarts. The HTTP API similarly must be started by the Vault server on start so that clients can interact with it.

Once started, the Vault is in a _sealed_ state. Before any operation can be performed on the Vault it must be unsealed. This is done by providing the unseal keys. When the Vault is initialized it generates an encryption key which is used to protect all the data. That key is protected by a master key. By default, Vault uses a technique known as Shamir's secret sharing algorithm to split the master key into 5 shares, any 3 of which are required to reconstruct the master key

## How to install vault on the machine

Installing the vault is very simple. Download latest available image on [vault page](https://www.vaultproject.io/downloads.html), find the appropriate package for your system and download it. Vault is packaged as a zip archive.

After downloading Vault, we unzip the package. Vault runs as a single binary named `vault`. Any other files in the package can be safely removed and Vault will still function.

The final step is to make sure that the `vault` binary is available on the `PATH`.

We can verify our installation via this command:

```bash
$ vault -v

Vault v1.0.3 ('85909e3373aa743c34a6a0ab59131f61fd9e8e43')
```

## Run the vault locally

### Run vault 'dev' mode on local machine

Firstly we can start Vault as a server in "dev" mode like so: `vault server -dev`. This dev-mode server requires no further setup, and our local `vault` CLI will be authenticated to talk to it. This makes it easy to experiment with Vault or start a Vault instance for development. Every feature of Vault is available in "dev" mode.

> Never use "dev" mode server in production. It is unsecured and will lose data on every restart since it stores data in-memory)

Start the vault. You should see output similar:

```bash
$ vault server -dev

==> Vault server configuration:
             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
                 Storage: inmem
                 Version: Vault v1.0.3
             Version Sha: 85909e3373aa743c34a6a0ab59131f61fd9e8e43
...
Unseal Key: d+QjDvYB9ys7+/BfOV1RdTHqg0LeSvurmpu6n4ZyWlw=
Root Token: s.E1qGaBVCSsykjjr4HttLa9E1
...
```

With the dev server running, do the following three things. Launch a new terminal session, where we export vault address: `export VAULT_ADDR='http://127.0.0.1:8200'`

> On Windows you must use command `set` instead `export`.

or if we want to use secure version `export VAULT_ADDR='https://127.0.0.1:8201'.`

We save the unseal key and root token copy and export too: `export VAULT_DEV_ROOT_TOKEN_ID="s.E1qGaBVCSsykjjr4HttLa9E1"`

Verify the server is running by running the `vault status` command.

### Run vault server on the local machine

We should have installed vault on your local machine, without any running instance of the vault. If we would like to start vault server we need to have the configuration file. Vault is configured using HCL files. The configuration file for Vault is relatively simple:

```hcl
// Enable UI
ui = true

// Filesystem storage
storage "file" {
  path = "./vault-volume"
}

// TCP Listener
listener "tcp" {
  address = "0.0.0.0:8201"
  tls_disable = "true"
}
```

Then we are able to run command to start vault server:

`vault server -config vault-config.hcl`

Initialization is the process configuring the Vault. This only happens once when the server is started against a new backend that has never been used with Vault before. During initialization, the encryption keys are generated, unseal keys are created, and the initial root token is setup. To initialize Vault use `vault operator init`. This is an *unauthenticated* request, but it only works on brand new Vaults with no data. Initialization outputs two incredibly important pieces of information: the _unseal keys_ and the _initial root token_. This is the **only time ever** that all of this data is known by Vault, and also the only time that the unseal keys should ever be so close together. Don’t forget these keys!

```bash
$ vault operator init

Unseal Key 1: XX3kfDDqWNKDbP31enN/3ZWgrIb3BLX/qJ02IBVYDShc
Unseal Key 2: 3kxl7fALXdlO5yQxBsY4J2V+ZHYX3fwuFogZY4Zn8mDZ
Unseal Key 3: 3sJPGEsZ7B789smpIV/POnbBp57fG8ay0k13zK6FGh5d
Unseal Key 4: 0cZE0C/gEk3YHaKjIWxhyyfs8REhqkRW/CSXTnmTilv+
Unseal Key 5: fYhZOseRgzxmJCmIqUdxEm9C3jB5Q27AowER9w4FC2Ck

Initial Root Token: s.KkNJYWF5g0pomcCLEmDdOVCW
```

Every initialized Vault server starts in the _sealed_ state. From the configuration, Vault can access the physical storage, but it can't read any of it because it doesn't know how to decrypt it. The process of teaching Vault how to decrypt the data is known as _unsealing_ the Vault.

If we lost these keys or get some errors during the process we must start again. Stop any running instance of the Vault. Delete whole files created by Vault ( vault-volume/core/, vault-volume/sys/, ...) and start again.

If we would like to *unseal* vault we must input three times any of 5 unseal keys:

```bash
vault operator unseal XX3kfDDqWNKDbP31enN/3ZWgrIb3BLX/qJ02IBVYDShc
vault operator unseal 3kxl7fALXdlO5yQxBsY4J2V+ZHYX3fwuFogZY4Zn8mDZ
vault operator unseal 3sJPGEsZ7B789smpIV/POnbBp57fG8ay0k13zK6FGh5d
```

After each command, we should see increasing `Unseal Progress    1/3`. After that we can check `vault status` and we should see `Sealed false`.

With the root token, we are able to log in like root `vault login s.KkNJYWF5g0pomcCLEmDdOVCW`.

and we can reseal the Vault with `vault operator seal.`

## Run vault on docker

### Run vault 'dev' on docker and use with Spring

Our prerequisites are to have installed and running docker (with docker-compose). The vault has the official repository [on docker hub](https://hub.docker.com/_/vault).

If we would like to start vault on docker without any other file:

`$ docker run --cap-add=IPC_LOCK -e 'VAULT_DEV_ROOT_TOKEN_ID=`00000000-0000-0000-0000-000000000000"`' -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:`8200`' vault`  

In our case we create `docker-compose.dev.yml` file, where we configured all necessary parameters for configurations:

```yaml
version: '3.6'
services:
  vault:
    image: vault:latest
    container_name: vault
    restart: on-failure:10
    ports:
      - "8201:8201"
    environment:
      VAULT_ADDR: 'https://0.0.0.0:8201'
      VAULT_LOCAL_CONFIG: '{"listener": [{"tcp":{"address": "0.0.0.0:8201","tls_disable":"0", "tls_cert_file":"/data/vault-volume/certificate.pem", "tls_key_file":"/data/vault-volume/key.pem"}}], "default_lease_ttl": "168h", "max_lease_ttl": "720h"}, "ui": true}'
      VAULT_DEV_ROOT_TOKEN_ID: '00000000-0000-0000-0000-000000000000'
      VAULT_TOKEN: '00000000-0000-0000-0000-000000000000'
    cap_add:
      - IPC_LOCK
    volumes:
      - vault-volume:/data
    healthcheck:
      retries: 5
    command: server -dev -dev-root-token-id="00000000-0000-0000-0000-000000000000"
    networks:
      - sk_cloud
```

For development purpose, we use very easy token  `00000000-0000-0000-0000-000000000000.`

Before we tried this command we should see, that we use `certificate.pem` and `key.pem` which were generated to have secured SSL communication between the running vault and other application, like Spring. These certificates were generated by [openssl](https://www.openssl.org/).

Finally, before you start this service, we need to have that certificate on that volume.

`In project root folder` we run following commands:

```bash
docker volume create vault-volume
docker run -v vault-volume:/data --name helper busybox true
docker cp ./vault-volume helper:/data
docker rm helper
```

We can run whole docker file with this command:

`docker-compose -f docker-compose.dev.yml up --build -d`

### Run vault server on docker

Our requirements are very similar to the previous case. In this scenario, we use `docker-compose.server.yml` file where are some changes. We moved the initial script to a separate file because the vault generates keys during startup and these keys are required for further configuration.

```yaml
version: '3.6'
services:

  vault:
    image: vault:latest
    container_name: vault
    restart: on-failure:10
    ports:
      - "8201:8201"
    environment:
      VAULT_ADDR: 'https://0.0.0.0:8201'
    cap_add:
      - IPC_LOCK
    volumes:
      - vault-volume:/data
    healthcheck:
      retries: 5
    command: ./workflow-vault.sh
    networks:
      - sk_cloud
```

This workflow-vault script is placed on vault-volume with certificates. We also need this configuration file from the previous step, `vault-test.hcl` where is defined where are placed the certificates and keys.

And we use `spring-policy.hcl` the file which is used for policy.

```hcl
path "kv/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "kv/my-secret" {
  capabilities = ["read"]
}
```

And the bash script `workflow-vault.sh`:

```bash
#!/usr/bin/env bash

# Start vault
vault server -config vault-test.hcl

# Export values
export VAULT_ADDR='https://0.0.0.0:8201'
export VAULT_SKIP_VERIFY='true'

# Parse unsealed keys
mapfile -t keyArray < <( grep "Unseal Key " < generated_keys.txt  | cut -c15- )

vault operator unseal ${keyArray[0]}
vault operator unseal ${keyArray[1]}
vault operator unseal ${keyArray[2]}

# Get root token
mapfile -t rootToken < <(grep "Initial Root Token: " < generated_keys.txt  | cut -c21- )
echo ${rootToken[0]} > root_token.txt

export VAULT_TOKEN=${rootToken[0]}

# Enable kv
vault secrets enable -version=1 kv

# Enable userpass and add default user
vault auth enable userpass
vault policy write spring-policy spring-policy.hcl
vault write auth/userpass/users/admin password=${SECRET_PASS} policies=spring-policy

# Add test value to my-secret
vault kv put kv/my-secret my-value=s3cr3t
```

Then we can add this generated token to Spring Application and use it.

## Use Vault with Spring Boot Application

With Spring Cloud Vault we can access our secrets inside Vault. Secrets are picked up at startup of our application. Spring Cloud Vault uses the data from your application (application name, active contexts) to determine contexts paths in which you stored your secrets. First, we add to our pom file this dependency:

```java
<dependency>
            <groupId>org.springframework.vault</groupId>
            <artifactId>spring-vault-core</artifactId>
            <version>${spring.vault.core.version}</version>
</dependency>
```

The current version of Spring Vault requires Spring Framework in version 4.3.7.RELEASE or better.

After that, we create Configuration class, where we defined our token to the vault, and where can find our vault instance ("vault" represent name from docker container, running on port 8201 which is exported to the world via `https`)

```java
@Configuration
public class VaultConfig extends AbstractVaultConfiguration {

    @Override
    public ClientAuthentication clientAuthentication() {
        return new TokenAuthentication("00000000-0000-0000-0000-000000000000");
    }

    @Override
    public VaultEndpoint vaultEndpoint() {
        return VaultEndpoint.create("vault", 8201);
    }
}
```

After that we create Credential Service class, where we are able to create, read and delete our credentials or secrets:

```java
@Service
public class CredentialsService {

    private VaultTemplate vaultTemplate;

    public void secureCredentials(String storagePlace, Credentials credentials) {
        initVaultTemplate();
        vaultTemplate.write("kv/" + storagePlace, credentials);
    }

    public Credentials accessCredentials(String nameOfsecrets) {
        initVaultTemplate();
        VaultResponseSupport<Credentials> response = vaultTemplate.read("kv/" + nameOfsecrets, Credentials.class);
        return response.getData();
    }

    public void deleteCredentials(String name) {
        initVaultTemplate();
        vaultTemplate.delete("kv/" + name);
    }
}
```

We use `kv` for storing our credentials, because is easy to reach them via Rest services, but before start Spring application we must say to vault that kv is enabled `vault secrets enable -version=1 kv`

And if we work with the Spring Boot certificate, we also need to know where the vault certificates are stored, so we have to tell them where they are: \```keytool -importcert -file /data/vault-volume/certificate.pem -alias vault -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit -noprompt` ``

All the above commands are stored into the script which is running in docker-compose along with vault service.

```yaml
...
  vault-java-demo:
    image: gitlabregistry.exxeta.com/exxetask/vault-java-demo:develop
    container_name: vault-java-demo
    restart: on-failure:10
    ports:
      - "8444:8444"
    volumes:
      - vault-volume:/data
    healthcheck:
      retries: 5
    environment:
      - JAVA_OPTS=-Xmx64m
    depends_on:
      - vault
    networks:
      - sk_cloud

```

## Check communication with Postman

If we have run the Vault and Spring application correctly, we can use any browser or [postman application](https://www.getpostman.com/) to verify. The Vault is can be accessed on https://localhost:8201 and Spring application using swagger https://localhost:8444/swagger-ui.html#/ where you can perform multiple rest services, such as adding, retrieving, or deleting credentials.


![Sequence_diagram](https://user-images.githubusercontent.com/17578897/171566786-55d87987-a4d5-4cae-9a3e-eb49e32d4837.png)

Figure 2: Sequence diagram of Vault and Spring Boot Application (Click to enlarge)

## Conclusion

Feel free give us feedback. We hope this tutorial has provided you with new information about the Vault and how to use it with Spring Framework.

tags: `Spring, Vault, Security, Docker, Docker-compose, OpenSSL, HashiCorp `

## Sources

- [HashiCorpVault page](https://www.vaultproject.io)
- [Docker vault image](https://hub.docker.com/_/vault)
- [Certificate generator, OpenSSL](https://www.openssl.org/)
- [Spring boot vault](https://docs.spring.io/spring-vault/docs/1.0.0.RELEASE/reference/html/)
