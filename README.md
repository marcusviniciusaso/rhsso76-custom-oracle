O objetivo deste documento é criar uma imagem customizada do Red Hat Single Sign-On 7.6 com driver Oracle para conexão com banco de dados.

# Criação de imagem

A imagem oficial do RH-SSO em seu estado natural não possui o driver do Oracle, portanto é necessário adicionar o driver à imagem base. É preciso realizar alguns passos antes da alteração da imagem em si.

**1. Atualizar os recursos do RH-SSO 7.6 no OpenShift**
```bash
for resource in sso76-image-stream.json \
  passthrough/sso76-https.json \
  passthrough/sso76-postgresql.json \
  passthrough/sso76-postgresql-persistent.json \
  reencrypt/ocp-4.x/sso76-ocp4-x509-https.json \
  reencrypt/ocp-4.x/sso76-ocp4-x509-postgresql-persistent.json
do
  oc replace -n openshift --force -f \
 https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/${resource}
done
```
Através do for acima ou executando cada laço do for individualmente:
```
oc replace -n openshift --force -f \
https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/sso76-image-stream.json

oc replace -n openshift --force -f \
https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/passthrough/sso76-https.json

oc replace -n openshift --force -f \
https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/passthrough/sso76-postgresql.json

oc replace -n openshift --force -f \
https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/passthrough/sso76-postgresql-persistent.json

oc replace -n openshift --force -f \
https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/reencrypt/ocp-4.x/sso76-ocp4-x509-https.json

oc replace -n openshift --force -f \
https://raw.githubusercontent.com/jboss-container-images/redhat-sso-7-openshift-image/sso76-dev/templates/reencrypt/ocp-4.x/sso76-ocp4-x509-postgresql-persistent.json
```

**2. Importar a imagem do catálogo da Red Hat**
```
oc -n openshift import-image rh-sso-7/sso76-openshift-rhel8:7.6 --from=registry.redhat.io/rh-sso-7/sso76-openshift-rhel8:7.6 --confirm
```
Esse passo serve para importar a imagem base do RH-SSO no OpenShift. Pode ser que esse comando não funcione em alguns ambientes, se não tiver acesso ao catálogo da Red Hat. Caso não funcione, verificar se no namespace openshift já existe um ImageStream do RH-SSO 7.6, se existir pode usar essa imagem como imagem base (FROM do Dockerfile).

Consultar ImageStream:
```
oc get is -n openshift
```
Consultar a tag do ImageStream:
```
oc get istag -n openshift
```

**3. Criar template customizado**

O template já está criado e pode ser reutilizado, mas caso em algum momento o template precise ser criado do zero novamente. Baixar o template padrão para um arquivo yaml:
```
oc process sso76-ocp4-x509-https SSO_ADMIN_USERNAME=<USERNAME> SSO_ADMIN_PASSWORD=<PASSWORD> -n openshift -o yaml > sso76-ocp4-x509-https.yaml
```
Esse template precisa ser adaptado para a conexão com o Oracle. Utilizar o arquivo [template-customizado-sso76-ocp4.yaml](INSERIR LINK) como uma versão final do template.

**4. Habilitar rota para o registry interno do OpenShift**

Para utilizar a imagem base do registry interno do OpenShift, deve habilitar a rota para o registry do OpenShift:
```
oc patch config.imageregistry cluster -n openshift-image-registry --type merge -p '{"spec":{"defaultRoute":true}}'
```

Se este comando não funcionar, por conta que o campo defaultRoute não existe, editar diretamente o yaml:
```
oc edit config.imageregistry cluster -n openshift-image-registry
```

Para pesquisar a rota para o registry interno:
```
oc get route -n openshift-image-registry
```
Essa rota será utilizada no FROM do Dockerfile com a imagem base. Uma alternativa é utilizar a imagem diretamente do catálogo da Red Hat:
> registry.redhat.io/rh-sso-7/sso76-openshift-rhel8:7.6

**5. Criar o Dockerfile**

```
FROM <URL_REGISTRY>/openshift/sso76-openshift-rhel8:7.6

COPY ojdbc8.jar /opt/eap/extensions/ojdbc8.jar
```

**6. Fazer pull da imagem base**

Pull do registry interno:
```bash
podman pull <URL_REGISTRY>/openshift/sso76-openshift-rhel8:7.6
```

Ou diretamente do catálogo da Red Hat:
```bash
podman pull registry.redhat.io/rh-sso-7/sso76-openshift-rhel8:7.6
```

**7. Fazer o build da imagem**
```bash
podman build -f Dockerfile -t localhost/sso76-custom-oracle:1.0 .
```

**8. Fazer o login no registry**

Registry interno:
```bash
podman login -u USUARIO <URL_REGISTRY>
```

Ou em outro registry, como o quay:
```bash
podman login -u USUARIO quay.io
```

**9. Fazer o push da imagem**

Registry interno:
```bash
podman push localhost/sso76-custom-oracle:1.0  <URL_REGISTRY>/openshift/sso76-custom-oracle:1.0
```

Ou em outro registry, como o quay:
```bash
podman push localhost/sso76-custom-oracle:1.0  quay.io/USUARIO/sso76-custom-oracle:1.0
```

## Instalação do RH-SSO

Uma vez criada a imagem e disponibilizada no registry, a instalação pode ser realizada.

**1. Criar namespace onde será instalado o RH-SSO**
```
oc new-project rhsso
```

**2. Editar o template para colocar a imagem no campo image**

Editar a linha 190 do template [template-customizado-sso76-ocp4.yaml](INSERIR LINK) para colocar o endereço, nome e tag da imagem.

**3. Criação da secret com os dados de acesso ao banco de dados**

Essa secret é utilizada pelo template para disponibilizar variáveis de ambiente para o RH-SSO.
```
oc create secret generic sso-database-secret --from-literal db-username=USUARIO --from-literal db-password=SENHA --from-literal db-driver-name=oracle --from-literal db-driver=oracle.jdbc.OracleDriver --from-literal db-xa-driver=oracle.jdbc.xa.client.OracleXADataSource --from-literal db-eap-module=com.oracle --from-literal db-driver-file="/opt/eap/extensions/ojdbc8.jar" --from-literal db-jdbc-url="JDBC_URL" -n rhsso
```

**4. Deploy do RH-SSO**
```
oc create -f template-customizado-sso76-ocp4.yaml -n rhsso
```

Para refazer o deploy, basta repetir o passo 4 da instalação e o passo 3 se houver alteração da conexão com o banco de dados.