#Métricas e Dashboard para uso produtivo - RHBK

## Opção 1: O Caminho Nativo (OpenShift User-Workload Monitoring)


1.Habilitamos o recurso de User-Workload Monitoring no cluster ROSA (geralmente é só criar um ConfigMap no namespace openshift-user-workload-monitoring).
    - Aplicar manifesto 00-enableUserWorkload.yaml
2.Criamos um arquivo YAML chamado ServiceMonitor no namespace rhbk. Esse arquivo diz ao Prometheus do OpenShift: "Vai no Keycloak a cada 30 segundos e colete as métricas".
    - Aplicar manifesto 01-serviceMonitor.yaml

3.Para visualizar, vocês podem usar a própria aba "Observe -> Dashboards" nativa do painel do OpenShift, ou instalar o Grafana Operator no cluster para importar um dashboard pronto da comunidade do Keycloak.

4. Em "Observe -> Targets", procure por rhbk e valide se o serviço de métricas está UP.

### Principais métricas para usar em ambiente produtivo com o RHBK

1. Saúde do Banco de Dados:
Erros de timeout de banco, pool de conexões utilizar as metricas a seguir.
    Conexões Ativas (Em uso):
        agroal_active_count{namespace="rhbk"}

    Fila de Espera (Atenção Máxima!): Se isso aqui sair do zero, significa que requisições estão esperando o banco liberar espaço para serem atendidas. É o alerta vermelho do seu painel:
        agroal_pending_count{namespace="rhbk"}

2. Saúde da Aplicação (JVM e Sistema)
Como o Keycloak roda em Java, vazamentos de memória ou estouro de CPU são os maiores causadores de lentidão.

    Uso de Memória RAM (Heap):
    sum(jvm_memory_used_bytes{namespace="rhbk", area="heap"}) by (pod) / 1024 / 1024 (Resultado em MB)

    Uso de CPU do Processo:
    process_cpu_usage{namespace="rhbk"}

    Picos de Garbage Collection: Se o GC estiver demorando muito, o Keycloak "congela" por alguns segundos.
    rate(jvm_gc_pause_seconds_sum{namespace="rhbk"}[5m])

3. Tráfego e Performance HTTP
Medir fluxo de usuários na porta do seu servidor.

    Taxa de Requisições (RPS): Quantas requisições por segundo seu Keycloak está recebendo:
    sum(rate(http_server_requests_seconds_count{namespace="rhbk"}[5m]))

    Erros de Servidor (5xx): Se esse gráfico subir, o Keycloak está quebrando (ex: banco caiu, erro de código):
    sum(rate(http_server_requests_seconds_count{namespace="rhbk", status=~"5.."}[5m]))

    Latência (Tempo de Resposta): O tempo médio (em segundos) que o Keycloak leva para responder:
    rate(http_server_requests_seconds_sum{namespace="rhbk"}[5m]) / rate(http_server_requests_seconds_count{namespace="rhbk"}[5m])

4. Métricas de Negócio (Logins e Tokens)
Para medir o sucesso do seu Identity Provider, obsevar os "caminhos" (URIs) de login.

    Emissão de Tokens (Logins com sucesso): Conta quantas vezes o endpoint de token foi chamado:
    sum(rate(http_server_requests_seconds_count{namespace="rhbk", uri=~".*/protocol/openid-connect/token"}[5m]))



