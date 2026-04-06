# Procedimento de como configurar cognito

Acesse o Amazon Cognito e clique no nome do User Pool que você criou.
Na aba principal (a tela de "Overview" ou "Visão geral"), procure por User pool ID (ID do grupo de usuários). Vai ser algo como us-east-1_aBcD123.
Anote em qual região da AWS você está (ex: us-east-1, sa-east-1 para São Paulo, etc).
Monte a URL colando essas duas informações neste padrão:

https://cognito-idp.<sua-regiao>.amazonaws.com/<seu-user-pool-id>

Exemplo: Se sua região é us-east-1 e o ID é us-east-1_aBcD123, sua Issuer URL será:
https://cognito-idp.us-east-1.amazonaws.com/us-east-1_aBcD123


Criar um user pool  e copiar o user pool id

- us-east-2_vEPHj3xRr

Região 

us-east-2

Final:
https://cognito-idp.us-east-2.amazonaws.com/us-east-2_vEPHj3xRr





Client ID
5cujninn0srqmhqmoidrfd9sm5
Client secret
904d7jlf9ufff827gna5aijscgnjsqflkbfvndl49d64ng30211

URL de CallBack:

https://oauth-openshift.apps.<nome-cluster>.<id>.p1.openshiftapps.com/oauth2callback/cognito-idp

https://oauth-openshift.apps.rosa-f8plh.8guc.p3.openshiftapps.com/oauth2callback/cognito-idp


rosa create idp --cluster=rosa-f8plh\
  --type=openid \
  --name=cognito-aws \
  --client-id=5cujninn0srqmhqmoidrfd9sm5 \
  --client-secret=904d7jlf9ufff827gna5aijscgnjsqflkbfvndl49d64ng30211 \
  --issuer-url=https://oauth-openshift.apps.rosa-f8plh.8guc.p3.openshiftapps.com/oauth2callback/cognito-idp \
  --email-claims=email \
  --username-claims=email


## OUTPUT

rosa create idp --cluster=rosa-f8plh\
  --type=openid \
  --name=cognito-aws \
  --client-id=5cujninn0srqmhqmoidrfd9sm5 \
  --client-secret=904d7jlf9ufff827gna5aijscgnjsqflkbfvndl49d64ng30211 \
  --issuer-url=https://oauth-openshift.apps.rosa-f8plh.8guc.p3.openshiftapps.com/oauth2callback/cognito-idp \
  --email-claims=email \
  --username-claims=email
I: Configuring IDP for cluster 'rosa-f8plh'
I: Identity Provider 'cognito-aws' has been created.
   It may take several minutes for this access to become active.
   To add cluster administrators, see 'rosa grant user --help'.

I: Callback URI: https://oauth.rosa-f8plh.8guc.p3.openshiftapps.com:443/oauth2callback/cognito-aws
I: To log in to the console, open https://console-openshift-console.apps.rosa.rosa-f8plh.8guc.p3.openshiftapps.com and click on 'cognito-aws'.


rosaLogin123@

rosaLogin123@@






   oc login https://api.rosa-f8plh.8guc.p3.openshiftapps.com:443 --username cluster-admin --password 4Hvij-cQTxd-kiGgS-ubmDF





# Passo a passo de criação do Cognito como Identity Provider


## Criar e configurar um Cognito User Pool
0. Variavel de ambiente   
1. Criar um user pool
2. Criar um user pool domain
3. Criar usuários no user pool
4. Criar um app client para o user pool
5. Configurar OpenShift OAuth para usar o app client

## 0. Variaveis de ambiente
export GUID=f8plh
export CLIENTE="Inc Corp"

## 1. Criando um user pool
Create and Configure a Cognito User Pool
To set up the Amazon Cognito service we need to do a few things:

```
aws cognito-idp create-user-pool --pool-name rosa-${GUID} --auto-verified-attributes email \
  --admin-create-user-config '{"AllowAdminCreateUserOnly": true}' | jq . 
```
  
Exporte o AWS_USER_POOL_ID

``` 
export AWS_USER_POOL_ID=$(aws cognito-idp list-user-pools --max-results 1 | jq -r '.UserPools[0].Id')

echo ${AWS_USER_POOL_ID}
Sample Output
us-east-2_ABCDE1234
``` 

## 2. Criar um user pool domain

Agora crie o domínio para Cognito user pool.

export DOMAIN=redhat.com

```
aws cognito-idp create-user-pool-domain \
  --domain "rosa-${GUID}" \
  --user-pool-id "${AWS_USER_POOL_ID}"
```

O próximo passo é criar usuários no user pool e designar como cluster-admins e usuários restritos.

## 3. Criar usuários no user pool

```
aws cognito-idp admin-create-user \
  --user-pool-id "${AWS_USER_POOL_ID}" \
  --username admin \
  --temporary-password "JoeDoedaSilva-A2@26" \
  --user-attributes Name=name,Value="Cluster Admin ${CLIENTE}" Name="email",Value="admin@${DOMAIN}" Name="email_verified",Value="true" \
  --message-action SUPPRESS
``` 


Crie alguns usuários restritos adicionais:

```
aws cognito-idp admin-create-user \
  --user-pool-id "${AWS_USER_POOL_ID}" \
  --username user1 \
  --temporary-password "JoeDoedaSilva-A2@26" \
  --user-attributes Name=name,Value="Usuario Dev 1" Name="email",Value="user1@${DOMAIN}" Name="email_verified",Value="true" \
  --message-action SUPPRESS


aws cognito-idp admin-create-user \
  --user-pool-id "${AWS_USER_POOL_ID}" \
  --username user1 \
  --temporary-password "JoeDoedaSilva-A2@26" \
  --user-attributes Name=name,Value="Usuario Dev 2" Name="email",Value="user2@${DOMAIN}" Name="email_verified",Value="true" \
  --message-action SUPPRESS
```

Determinar a URL de CallBack

```
CLUSTER_DOMAIN=$(rosa describe cluster -c "rosa-${GUID}" -o json | jq -r '. | .domain_prefix + "." + .dns.base_domain')

echo "OAuth callback URL: https://oauth.${CLUSTER_DOMAIN}:443/oauth2callback/Cognito"


```

Tome nota da URL de resultado que será utilizada em seguida.

## 4. Criar um app client para o user pool
Agora criar um app client no Amazon Cognito. 

```
aws cognito-idp create-user-pool-client \
  --user-pool-id $AWS_USER_POOL_ID \
  --client-name rosa-${GUID} \
  --generate-secret \
  --supported-identity-providers COGNITO \
  --callback-urls "https://oauth.${CLUSTER_DOMAIN}:443/oauth2callback/Cognito" \
  --allowed-o-auth-scopes "phone" "email" "openid" "profile" \
  --allowed-o-auth-flows code \
  --allowed-o-auth-flows-user-pool-client

``` 

Salve o ClientID e o ClientSecret em variáveis de ambiente:

``` 
export AWS_USER_POOL_CLIENT_ID=$(aws cognito-idp list-user-pool-clients --user-pool-id $AWS_USER_POOL_ID | jq -r '.UserPoolClients[0].ClientId')

export AWS_USER_POOL_CLIENT_SECRET=$(aws cognito-idp describe-user-pool-client --user-pool-id $AWS_USER_POOL_ID --client-id ${AWS_USER_POOL_CLIENT_ID} | jq -r '.UserPoolClient.ClientSecret')
```

## 5. Configurar OpenShift OAuth para usar o app client

Configure o identity provider no OpenShift:

```
rosa create idp \
--cluster rosa-${GUID} \
--type openid \
--name Cognito \
--client-id ${AWS_USER_POOL_CLIENT_ID} \
--client-secret ${AWS_USER_POOL_CLIENT_SECRET} \
--issuer-url https://cognito-idp.$(aws configure get region).amazonaws.com/${AWS_USER_POOL_ID} \
--email-claims email \
--name-claims name \
--username-claims username

```

Aguarde de 2 a 5 minutos para que o provedor apareça na tela inicial do console web do OpenShift.
Siga as instruções de como efetuar o login na console. 

Agora vamos dar as permissões de cluster admin para o usuário admin criado no Amazon Cognito.

Logue-se com usuário cluster-admin do seu cluster OpenShift para executar os próximos passos.

```
oc adm policy add-cluster-role-to-user cluster-admin admin
```

## Finalizando

Entre na console web, faça logout do usuário atual e selecione o botão Cognito. Execute o login  com usuário e senha criados para seu usuário na interface.

Por questões de segurança é recomendável remover o usuário temporário cluster-admin criado no idp padrão do cluster.

```
rosa delete admin -c rosa-${GUID} -y
```

Você pode remover o usuário cluster-admin e seus objetos associados:
```
oc delete user cluster-admin
oc delete identity cluster-admin:cluster-admin
```

[^note]:
 para remover o cluster-admin você deve estar logado com o usuário com poderes de cluster-admin do idp Cognito


