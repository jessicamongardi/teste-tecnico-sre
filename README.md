# Teste Técnico – Site Reliability Engineer (SRE)

## 01 - Descreva um incidente em ambiente de produção que você tenha resolvido. Informe qual era o sintoma, como você o investigou e qual foi a causa raiz.

### Resposta

Um dos incidentes mais relevantes em que atuei envolveu uma plataforma composta por diversos microserviços, onde usuários começaram a reportar inconsistências na visualização de informações de auditoria. Embora as transações fossem processadas normalmente, parte dos registros deixava de aparecer nas consultas realizadas pelas áreas de negócio.

Como o problema impactava a rastreabilidade das operações, a prioridade inicial foi entender se a falha estava na aplicação, na coleta dos eventos ou na plataforma de observabilidade.

A investigação começou pelo mapeamento completo do fluxo dos dados. Foram analisados os microserviços responsáveis pela geração dos eventos, o Filebeat utilizado na coleta dos logs e o Elasticsearch responsável pelo armazenamento das informações.

Durante as análises, identificamos que determinados eventos eram gerados pelas aplicações, mas não chegavam corretamente ao Elasticsearch. Inicialmente realizamos ajustes nos parâmetros de coleta do Filebeat, alterando intervalos de leitura e configurações relacionadas ao processamento dos arquivos. Apesar das melhorias, o problema continuou ocorrendo de forma intermitente.

Aprofundando a investigação, identificamos que um dos serviços responsáveis pelo processamento assíncrono de auditoria utilizava uma biblioteca que gerava volumes de logs significativamente acima do esperado. Esse comportamento causava impacto no fluxo de coleta e indexação dos eventos, reduzindo a confiabilidade das informações disponíveis para consulta.

Após validar a causa em homologação, realizamos os ajustes necessários e implantamos a correção em produção. O comportamento voltou à normalidade e os eventos passaram a ser coletados e indexados corretamente.

Além da resolução do incidente, conduzimos uma análise pós-incidente para identificar oportunidades de melhoria. Como resultado, foram criados novos alertas para monitoramento da saúde da pipeline de observabilidade, revisados limites operacionais e documentados procedimentos para acelerar futuras investigações.

Essa experiência reforçou a importância de tratar a observabilidade como um serviço crítico da plataforma, uma vez que a ausência de visibilidade pode comprometer significativamente a capacidade de resposta durante incidentes.

---

## 02 - Imagine que você estruturará a observabilidade do zero em múltiplos ambientes (desenvolvimento, homologação e produção) aplicados ao Kubernetes. Quais serão os seus primeiros passos?

### Resposta

Ao estruturar a observabilidade de um ambiente Kubernetes do zero, minha preocupação inicial não seria apenas escolher ferramentas, mas entender quais serviços são críticos para o negócio e quais indicadores representam a experiência real dos usuários.

A partir desse entendimento, definiria SLIs (Service Level Indicators) e SLOs (Service Level Objectives) para os principais serviços da plataforma. Esses indicadores serviriam como base para monitoramento, alertas e tomada de decisão durante incidentes.

Com os objetivos definidos, construiria a estratégia de observabilidade sobre três pilares principais: métricas, logs e rastreamento distribuído.

Como padrão de instrumentação, adotaria OpenTelemetry. Além de ser uma tecnologia amplamente utilizada no mercado, ela oferece flexibilidade e reduz o acoplamento entre aplicações e ferramentas específicas de observabilidade.

Na camada de métricas, utilizaria Prometheus para coleta dos dados e Grafana para visualização. O foco inicial seria monitorar indicadores relacionados aos chamados Golden Signals:

* Latência
* Tráfego
* Taxa de erros
* Saturação

Além disso, monitoraria componentes do Kubernetes, como nodes, pods, deployments, utilização de CPU, memória e disponibilidade dos serviços.

Para centralização de logs, utilizaria Elastic Agent, Elasticsearch e Kibana, permitindo correlação entre eventos de infraestrutura, aplicações e componentes da plataforma.

Na camada de rastreamento distribuído, implementaria OpenTelemetry Collector e Jaeger para acompanhar o fluxo completo das requisições entre os microserviços e reduzir o tempo necessário para identificação de gargalos e falhas.

Também considero importante incorporar observabilidade de segurança desde o início. Nesse cenário, utilizaria Wazuh integrado ao Elasticsearch para monitoramento de eventos de segurança, vulnerabilidades, alterações em arquivos críticos e requisitos de compliance.

Após a implementação técnica, dedicaria atenção especial à construção de dashboards orientados ao negócio, runbooks operacionais e alertas acionáveis. Meu objetivo seria evitar alertas excessivos e priorizar eventos que representem impacto real aos usuários.

Por fim, estabeleceria processos de revisão periódica dos indicadores e condução de postmortems para promover aprendizado contínuo e evolução da confiabilidade dos serviços.

### Arquitetura proposta

```text
                         +------------------+
                         |     Usuários     |
                         +--------+---------+
                                  |
                                  v
                       +---------------------+
                       | Kubernetes Cluster  |
                       +---------------------+
                                  |
             +--------------------+--------------------+
             |                    |                    |
             v                    v                    v

     +---------------+   +----------------+   +----------------+
     | Aplicações    |   | Nodes/Pods     |   | Wazuh Agents   |
     +-------+-------+   +--------+-------+   +--------+-------+
             |                    |                    |
             +--------------------+--------------------+
                                  |
                                  v

                   +----------------------------+
                   | OpenTelemetry Collector    |
                   +-------------+--------------+
                                 |
                +----------------+----------------+
                |                                 |
                v                                 v

        +---------------+                 +--------------+
        | Prometheus    |                 |    Jaeger    |
        +-------+-------+                 +--------------+
                |
                v

        +---------------+
        | Alertmanager  |
        +-------+-------+
                |
                v

        +---------------+
        |   Grafana     |
        +---------------+

                ^
                |
      +----------------------+
      | Elasticsearch        |
      +----------+-----------+
                 ^
                 |
      +----------+-----------+
      | Elastic Agent        |
      +----------+-----------+
                 ^
                 |
      +----------+-----------+
      | Wazuh Manager         |
      +----------+-----------+
                 |
                 v

            +---------+
            | Kibana  |
            +---------+
```

---

## 03 - Como você definiria os SLIs e os SLOs de um serviço crítico? Cite um exemplo e explique o que é o Error Budget.

### Resposta

SLI (Service Level Indicator) é uma métrica utilizada para medir a qualidade real de um serviço sob a perspectiva do usuário.

Alguns exemplos comuns incluem disponibilidade, latência, taxa de erros e sucesso das transações.

Já o SLO (Service Level Objective) representa a meta que desejamos atingir para determinado indicador. Ele funciona como um compromisso interno de confiabilidade e ajuda a direcionar prioridades técnicas e operacionais.

Como exemplo, considere uma API responsável pelo processamento de pagamentos.

Podemos definir:

**SLI:** Disponibilidade do serviço

**SLO:** 99,95% de disponibilidade mensal

Também podemos acompanhar:

**SLI:** Tempo de resposta

**SLO:** 95% das requisições respondidas em até 300 ms

A partir desses objetivos surge o conceito de Error Budget.

Se o SLO de disponibilidade é de 99,95%, existe uma margem aceitável de 0,05% de indisponibilidade. Em um mês com aproximadamente 43.200 minutos, isso representa cerca de 21 minutos de indisponibilidade permitida.

O Error Budget é importante porque cria equilíbrio entre inovação e estabilidade. Enquanto existe orçamento disponível, a equipe pode continuar realizando mudanças e evoluções normalmente. Quando esse orçamento começa a ser consumido rapidamente, torna-se necessário priorizar ações voltadas à confiabilidade antes de aumentar o volume de mudanças em produção.

Na prática, o Error Budget ajuda a transformar decisões técnicas em decisões orientadas por dados, reduzindo conflitos entre velocidade de entrega e estabilidade operacional.

---

## 04 - Relate uma atividade operacional à qual você aplicou automação com o intuito de reduzir o trabalho manual (toil): descreva a tarefa, as ferramentas utilizadas e o retorno obtido.

### Resposta

Uma atividade que gerava bastante esforço operacional era o suporte recorrente aos processos de build, deploy e correção de falhas em pipelines utilizados por dezenas de aplicações.

Com o crescimento da plataforma, tornou-se comum receber solicitações relacionadas a configurações incorretas de ambientes, problemas em Jenkinsfiles, dependências desatualizadas e inconsistências entre os ambientes de homologação e produção.

Além de consumir tempo da equipe, essas atividades tinham baixo valor agregado e se encaixavam no conceito de toil, por serem repetitivas, manuais e frequentemente reativas.

Para reduzir esse cenário, participei da padronização das bibliotecas compartilhadas utilizadas pelos pipelines e da automação de diversas validações executadas durante o processo de build e deploy.

Foram utilizadas ferramentas como Jenkins, GitLab, Docker, Kubernetes, Nexus e SonarQube. Diversas verificações passaram a ocorrer automaticamente, incluindo validações de configuração, análise de qualidade de código e identificação de dependências incorretas antes da publicação das aplicações.

Também revisamos pipelines legados e eliminamos referências para ambientes descontinuados, configurações obsoletas de proxy e apontamentos incorretos que geravam falhas recorrentes.

Como resultado, houve redução significativa das intervenções manuais da equipe, aumento da confiabilidade dos processos de implantação e menor tempo de recuperação quando ocorria algum problema.

Além do ganho operacional, a automação permitiu que a equipe direcionasse mais tempo para iniciativas de melhoria da plataforma, observabilidade e confiabilidade dos serviços, reduzindo a dependência de atividades repetitivas do dia a dia.

---

## 05 - De que maneira você utiliza a Inteligência Artificial em seu cotidiano profissional atualmente?

### Resposta

Utilizo Inteligência Artificial diariamente como uma ferramenta de apoio para acelerar análises técnicas, pesquisas e atividades operacionais.

Durante investigações de incidentes, costumo utilizá-la para validar hipóteses, analisar possíveis causas raiz e acelerar a navegação por documentações técnicas. Em ambientes complexos, essa abordagem ajuda a reduzir o tempo necessário para chegar a um direcionamento inicial de troubleshooting.

Também utilizo IA para revisão de scripts, elaboração de consultas para Elasticsearch, criação de dashboards de observabilidade, construção de regras de monitoramento e revisão de pipelines CI/CD.

Outra aplicação frequente é na produção de documentação técnica, incluindo runbooks, procedimentos operacionais, postmortems e diagramas de arquitetura.

Além do aspecto operacional, utilizo a IA como ferramenta de aprendizado contínuo para explorar novas tecnologias, padrões arquiteturais e práticas relacionadas a observabilidade, confiabilidade e engenharia de plataforma.

Acredito que a principal contribuição da IA atualmente é permitir que profissionais dediquem mais tempo à análise crítica, tomada de decisão e resolução de problemas complexos, reduzindo o esforço gasto em tarefas repetitivas e pesquisas iniciais.
