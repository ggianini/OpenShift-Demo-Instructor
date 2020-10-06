# OpenShift-Demo

### Demos

  * [Repositorio com varias demos](https://github.com/redhat-developer-demos)
  * [Deploy](https://docs.google.com/presentation/d/1rtbk08RyuVxkFHhfMhAFb6FooMWLXrwDmhgLgcG8ML8)

## Opcionais

** Mostrar cloud.redhat.com
* Billing, Insights, Ansible, etc.

** Update
** Mostrar provisionamento de node (ao final se der tempo)
** Storage

User: cost-user
Password: cost4User

## 0) Criar usuário admin

[Configuring an HTPasswd identity provider](https://docs.openshift.com/container-platform/4.5/authentication/identity_providers/configuring-htpasswd-identity-provider.html)

** Criar Identity Provider HTPasswd
* Administration → Cluster Settings → Global Configuration → OAuth

** Criar arquivo com as credenciais
```sh
htpasswd -c -B -b users.htpasswd ggianini redhat
```

** Acessar console

* Explicar linha de comando OC

```sh
oc get route -n openshift-console
```

[Using RBAC to define and apply permissions](https://docs.openshift.com/container-platform/4.5/authentication/using-rbac.html)

```sh
oc adm policy add-cluster-role-to-user cluster-admin ggianini
```

## 1) Criar código da app e colocar no github

* criar um novo repositório no Git

adicionar o arquivo index.php

```php
<?php
echo "<h1>Olá Mundo v1.0</h1> ";
echo $_SERVER['SERVER_ADDR']
?>
```

## 2) Criar app no Openshift

* Mostrar em que Node do cluster que ela foi provisionada
* Mostrar metricas, log, eventos, variaveis de ambiente, etc

## 3) Escalar app

* Mostrar novamente em que Node do cluster que ela foi provisionada
* Mostrar o balanceamento de carga
 * via curl
```bash
while [ true ]; do curl http://myphp-php-demo.apps.ocp.acme.com/; sleep 1; echo; done
```

```
oc get po -o wide
```
* Mostrar as metricas agregadas:
* Mostrar pod isolation: **remove do balanceamento!**

> escolher um POD, editar o YAML e mudar o label `deploymentconfig`

* Tentar matar dois containers da aplicação
 * via CLI
```
oc delete pod --force <id1> <id2>
```

* Escalar para 1 instancia

## 4) Criar um banco de dados

* MySQL
* Criar DB no openshift

```
       Username: redhat
       Password: redhat
  Database Name: sampledb
 Connection URL: mysql://mysql:3306/
```


 * Alterando autenticação
```
mysql -u root
ALTER USER 'redhat' IDENTIFIED WITH mysql_native_password BY 'redhat';
```

 * Importar a tabela sql

```
mysql -u redhat -h 127.0.0.1 -p

SHOW DATABASES;
USE sampledb;
SHOW TABLES;

CREATE TABLE cidade (id INT NOT NULL, nome VARCHAR(50) NOT NULL, PRIMARY KEY (id));

SHOW TABLES;
DESCRIBE cidade;

INSERT INTO cidade (id,nome) VALUES(1,"Rio de Janeiro");
INSERT INTO cidade (id,nome) VALUES(2,"Brasilia");
INSERT INTO cidade (id,nome) VALUES(3,"Vitoria");

SELECT * FROM cidade;
```
## Criando imagem personalizada no Catálogo

```
oc create -f meucrd.yaml
```

## 5) Webhook e Rsync

### 5.1) Webhook

 * Configurar código fonte do DB
```
<?php
echo "<h1 style='color:blue;'>OpenShift Demo v2.0</h1> ";
echo $_SERVER['SERVER_ADDR'];

echo "<br><hr>";
echo "<h2>Cidades cadastradas no Banco de Dados:</h2>";

$conn = new mysqli("mysql", "redhat", "redhat", "sampledb");
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$result = $conn->query("SELECT nome FROM cidade");

if ($result->num_rows > 0) {
    while($row = $result->fetch_assoc()) {
        echo "<h3>" . $row["nome"] . "</h3>";
    }
} else {
    echo "0 results";
}
$conn->close();
?>
```

### 5.2) Rsync

 * Permite copiar arquivos diretamente via rsync para dentro do Container (POD). Não recomendado devido a natureza efêmera do POD!

 * git clone localmente e fazer alterações

git clone --single-branch --branch minhabranch https://github.com/ggianini/minhademo.git


```
    oc rsync --exclude '.git' . pod-id:/opt/app-root/src --no-perms -w
```

## 6) Health Check e Debug

* Escalar para 2 instancias

```
while [ true ]; do curl http://myphp-php-demo.apps.ocp.acme.com/; sleep 1; echo; done
```
> The two probes have two separate purposes and start running as soon as the container is started

### 6.1 Liveness

> Liveness means your container is running, and if it fails then the container should be restarted.

* Criar pagina liveness.php

```
<?php
$filename = '/tmp/liveness';

if (file_exists($filename)) {
    header("HTTP/1.1 500 Internal Server Error");
} else {
        echo "Estou vivo!";
        echo "<br><hr>";
        echo "Essa mensagem significa que a aplicação está saudável.";
        echo "<br><hr>";
        echo "Caso essa URL devolva um erro 500, significa que o liveness falhou. Nesse caso o OpenShift irá deletar o pod";
}
?>

```

### 6.2 readiness

> Readiness means your service is ready to receive requests, so should be added to the service rotation.

* Criar a pagina readiness.php

```
<?php
$filename = '/tmp/readiness';

if (file_exists($filename)) {
    header("HTTP/1.1 500 Internal Server Error");
} else {
        echo "Estou pronto!";
        echo "<br><hr>";
        echo "Essa mensagem significa que a aplicação está pronta para ser inserida no balanceamento.";
        echo "<br><hr>";
        echo "Caso essa URL devolva um erro 500, significa que o liveness falhou. Nesse caso o OpenShift irá remover a aplicação do balanceador.";
}
?>
```
```
oc set probe dc/meu-app --readiness --get-url=http://:8080/readiness.php
```

Ou via interface gráfica:

/readiness.php
8080
3 1 null 10 1

/liveness.php
8080
3 1 20 10 1

```
oc set probe dc/meu-app --initial-delay-seconds=20 --liveness --get-url=http://:8080/liveness.php
```

* Fazer debug do container com readiness
 * acessar um dos PODs

```
oc rsh <pod id>

> touch /tmp/readiness
```

 * observar Página Overview do projeto na console do ocp

 * Criar /tmp/liveness

## 7) Idle resources

* Parar o `while [ true ]; do curl ...` - INATIVIDADE!
 * oc idle app
 * observar status dos PODs na console
 * Acessar a URL da app

## 8) Resource Quotas

* Escalar para 1
* Colocar limit e request de memoria e cpu

 ```
oc set resources dc/meu-app --limits=cpu=200m,memory=100Mi --requests=cpu=200m,memory=100Mi
```

* criar novo arquivo forkbomb.php

```php
<?php
$output = shell_exec("while :; do _+=( $((++__)) ); done");
?>
```
* Acessar url do arquivo forkbomb.php
* Acompanhar uso de recursos via console

 * Esperar ele matar o container
 * Observe as métricas na console do POD!

```
oc delete po <pod>
```

## 9) Auto scaling de deploymentconfig

* add auto scaler

```
oc autoscale dc/meu-app --min 1 --max 10 --cpu-percent=10
```

* inicia stress

```
oc run --generator=run-pod/v1 -it --rm load-generator --image=busybox /bin/sh
```
```
while true; do wget -q -O- <url>; done
```

* `oc delete hpa app`

## 10) Monitoramento e Logging

* Provisionar machineset m4.2xlarge

* Instalar Operators "elasticsearch" e "cluster logging", provisionar CRD com ZeroRedundancy e Node = 1

Vamos criar uma vista personalizada dos nossos logs, para isso posicione o cursor do mouse sobre os campos a seguir e clique em "add":

```
	Time
 hostname
 kubernetes.namespace_name
 kubernetes.container_name	
 message
```
```json
{
  "query": {
    "match": {
      "message": {
        "query": "server reached MaxRequestWorkers setting, consider raising the MaxRequestWorkers setting",
        "type": "phrase"
      }
    }
  }
}
```

** Mostrar provisionamento de node
