# OpenShift-Demo

### Demos

  * [Repositorio com varias demos](https://github.com/redhat-developer-demos)
  * [Deploy](https://docs.google.com/presentation/d/1rtbk08RyuVxkFHhfMhAFb6FooMWLXrwDmhgLgcG8ML8)
  * [MSA](https://cdn.rawgit.com/redhat-helloworld-msa/helloworld-msa)


## 1) Criar código da app e colocar no gogs

* criar um novo projeto no Git

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
The following service(s) have been created in your project: mysql.

       Username: redhat
       Password: redhat
  Database Name: sampledb
 Connection URL: mysql://mysql:3306/

For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/mysql-container/blob/master/5.7/README.md.
```

* Agrupar banco de dados com app

 * Port forward
```
	oc login ocp-master.example.com:8443
	oc port-forward <pod id> 3306:3306
```

 * Alterando autenticação
```
mysql -u root
ALTER USER 'redhat' IDENTIFIED WITH mysql_native_password BY 'redhat';
```

 * Importar a tabela sql

```
mysql -u admin -h 127.0.0.1 -p

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

## 5) Rsync e Webhook

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

```
    oc rsync --exclude '.git' . pod-id:/opt/app-root/src --no-perms -w
```
 * Stop rsync: `Ctrl + c`

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
    echo "";
    echo "Essa mensagem significa que a aplicação está saudável.";
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
    echo "";
    echo "Essa mensagem significa que a aplicação está pronta para ser balanceada.";
    echo "Caso essa URL devolva um erro 500, significa que o readiness falhou. Nesse caso o OpenShift irá retirar o pod do balanceador";
}
?>

oc set probe deploy/meu-app --readiness --get-url=http://:8080/readiness.php
oc set probe deploy/meu-app --initial-delay-seconds=20 --liveness --get-url=http://:8080/liveness.php


* git add and commit
* Adicionar health check pela console
 * `/readiness.php`
 * `/liveness.php`


* Fazer debug do container com readiness
 * acessar um dos PODs

```
oc rsh <pod id>

> touch /tmp/readiness
```

 * observar Página Overview do projeto na console do ocp

 * Criar /tmp/liveness

* Trocar versão para 3.0 no index.php

## 7) Idle resources

* Parar o `while [ true ]; do curl ...` - INATIVIDADE!
 * oc idle app
 * observar status dos PODs na console
 * Acessar a URL da app

## Deploy docker image

Add to Project > Deploy Image > Image Name: "sonatype/nexus3"

## 8) Resource Quotas

* Escalar para 1
* Colocar limit e request de memoria e cpu
 * `Application > Deployments > [dc id] > Actions > Edit Resource Limits`
 * cpu: 200m
 * memoria: 100m

* acessar o POD via terminal
 * bash shell bomb

```
while :; do _+=( $((++__)) ); done
```

* ssh no Node do OCP onde o POD foi provisionado

 ```
 docker ps | grep <pod id> | grep entrypoint
 docker stats <container id>
 ```

 * Esperar ele matar o container
 * Observe as métricas na console do POD!

```
oc delete po <pod>
```

## 9) Auto scaling

* add auto scaler pela console
  * `Application > Deployments > [dc id] > Actions > Add Autoscaler`
	 * min 1
	 * max 5
	 * CPU request 20%

* inicia stress test usando `ab`:
 * se precisar instalar

 ```
 sudo yum install httpd-tools
 ```

 * disparar a carga

 ```
 ab -q -n 100000 -c 50 <url>
 ```

* ou JMeter
 * Thread Group com 150 threads
 * Loop 670

* `oc delete hpa app`

## 10) Rollback

* Introduzir bug na app:

```
<?ph-
```

* Fazer rollback pela web console
 * `Applications > Deployment > [dc id] > History`
  * escolher um deployment anterior
  * clicar em Rollback!
  * marcar todas as opções

* Fazer rollback no git

```
git log --oneline -n5

git reset --hard <hash-do-commit-desejado>

 #ou
 #volta para o commit imediatamente anterior
git reset --hard HEAD~1

git push --force origin master

```

 * Iniciar deploy via DC manual!

## 11) A/B Testing

* Desagrupar o banco (opcional)
* Criar branch no git

```
git checkout -b novaversao
```

 * Alterar para versão 3.0
  * `git add, commit, push`

	* Adicionar nova versao da app ao mesmo projeto pela console do OCP
  * informar nome da branch

	```
	while [ true ]; do curl <url>; sleep 0.5; echo; done
	```

  * Editar a rota original (v1 da app) via console do OCP
  * Expandir `Advanced options`
   * em `Alternate Services`
   * marcar a opção `Split traffic across multiple services`
   * escolher o `service` da app v2
   * save

	* Escalar um pod para cada lado
	* Alterar a porcentagem de cada um

## 12) Blue Green Deployment

* Remover ab testing
* alterar as cores (blue, green)
* Editar a rota original (v1 da app) via console do OCP
 * escolher o `service` da app v2
 * save

* Remover versão 3

## 13) Security com ImageStream **<<<< PRECISA SER MAIS DETALHADO!!!!**

* Criar imagem docker do php
* Fazer push para o registry
* Criar image stream
* Criar o build config com base na imagem anterior
* Executar o build config

## 14) Jenkins CI/CD

### 14.1) Pipeline para app PHP **<<<< MELHOR CONVERTER ISSO NUM TEMPLATE (YAML OU JSON)**

```
node('maven') {
	def app = 'app'
	def cliente = 'cliente'
	def gitUrl = 'http://gogs-cicd-tools.apps.ocp.acme.com/gogsadmin/php-demo.git'

    stage 'Build image and deploy to dev'
    echo 'Building docker image'
    buildApp(app + '-dev', gitUrl, app)

    stage 'Deploy to QA'
    echo 'Deploying to QA'
    deploy(app + '-dev', app + '-qa', app)

    stage 'Wait for approval'
    input 'Aprove to production?'

    stage 'Deploy to Prod'
    deploy(app + '-dev', app, app)
}

def buildApp(String project, String gitUrl, String app){
    projectSet(project)

    sh "oc new-build php:5.6~${gitUrl} --name=${app}"
    sh "oc logs -f bc/${app}"
    sh "oc new-app ${app}"
    sh "oc expose service ${app} || echo 'Service already exposed'"
    sh "sleep 10"
}

def projectSet(String project) {
	sh "oc login --insecure-skip-tls-verify=true -u admin -p redhat@123 https://ocp-master.example.com:8443"
    sh "oc new-project ${project} || echo 'Project exists'"
    sh "oc project ${project}"
}

def deploy(String origProject, String project, String app){
    projectSet(project)
    sh "oc policy add-role-to-user system:image-puller system:serviceaccount:${project}:default -n ${origProject}"
    sh "oc tag ${origProject}/${app}:latest ${project}/${app}:latest"
    appDeploy(app)
}

def appDeploy(String app){
    sh "oc new-app ${app} -l app=${app} || echo 'Aplication already Exists'"
    sh "oc expose service ${app} || echo 'Service already exposed'"
}
```

* Criar `Build Config` manualmente

```
kind: "BuildConfig"
apiVersion: "v1"
metadata:
  name: "php-pipeline"
  annotations:
    pipeline.alpha.openshift.io/uses: '[{"name": "app", "namespace": "", "kind": "DeploymentConfig"}]'
spec:
  source:
    type: "Git"
    git:
      uri: "http://gogs-ci-cd.apps.example.com/gogsadmin/cliente.git"
  strategy:
    type: "JenkinsPipeline"
    jenkinsPipelineStrategy:
      jenkinsfilePath: ""
```

### 14.2) Pipeline para app JavaEE (Wildfly)

* **JavaEE/Maven Pipeline**: https://github.com/openshift/origin/tree/master/examples/jenkins/pipeline

* Criar um novo projeto pela console
* Add to project
 * Import YAML/template
 * Copiar o conteúdo do aquivo: https://github.com/openshift/origin/blob/master/examples/jenkins/pipeline/maven-pipeline.yaml
 * Create
 * Aguardar o provisionamento automático do Jenkins Server pela Console Overview Page
* Iniciar o Pipeline: `Builds > Pipelines > Start Pipeline`

* Faça um fork do projeto `https://github.com/openshift/openshift-jee-sample.git`
 * pode ser no Github ou no GOGs do OCP
* Faça uma alteração no fonte
 * `openshift-jee-sample/src/main/webapp/index.html`

```
...
<body>

 <section class='container'>
          <hgroup>
            <h1 style="color:blue;">Welcome to your WildFly application on OpenShift - CLIENTE!!!</h1>
          </hgroup>
...
```

 * `git add, commit, push`
* Inicie um novo Pipeline
* Abra a URL da app e veja o resultado

### 14.3) Pipeline para app JavaEE com Integration tests, Staging e Approval workflow

* criar novo projeto: `jboss-kitchensink-cicd`
* clonar o projeto `https://github.com/rafaeltuelho/jboss-kitchensink.git` no seu host local
* `Add to project`
 * copiar e colar o conteúdo do YAML: `jboss-kitchensink/osev3/JenkinsBuildConfig.yaml`
* Aguardar o Jenkins Server ser provisionado no projeto
 * Adicionar a role de admin ao serviceaccount `Jenkins` para permitir que o pipeline crie novos projetos (staging)
  ```
  oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:jboss-kitchensink-cicd:jenkins -n jboss-kitchensink-cicd
  ```
* Iniciar o pipeline
 * `Builds > Pipelines > Start Pipeline`
 * acompanhar o log da execução do pipeline

> **NOTE:**: faça o prepull das imagens usadas por este pipeline nos nodes do cluster

```
docker pull registry.access.redhat.com/jboss-eap-7/eap70-openshift
docker pull jboss/wildfly:10.0.0.Final
```


## Binary Build

oc new-build --binary --name=demo-swarm
oc start-build demo-swarm --from-dir=. --follow
oc new-app demo-swarm
oc expose demo-swarm

### Fabric8

```
mvn fabric8:deploy
```

## 15) Evacuate

```
oadm manage-node ocp-node01.example.com --evacuate --dry-run
```

## 16) Cockpit

* Acessar: https://ocp-master.example.com:9090

## 17) Cloud Forms

* Acessar: https://ocp-master.example.com:9090

## 18) Registry Console

* Ver url por meio do comando
```
oc get route -n default | grep -i registry
```
