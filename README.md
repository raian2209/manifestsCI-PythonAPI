# manifestsCI-PythonAPI

-----

# Repositório de Manifestos Kubernetes (GitOps)

Este repositório atua como a **única fonte da verdade** para o estado desejado da nossa aplicação no cluster Kubernetes. Ele contém todos os manifestos declarativos necessários para a implantação e é gerenciado através de um fluxo de trabalho GitOps.

## 🏛️ Papel na Arquitetura

Este projeto faz parte de uma arquitetura de CI/CD de dois repositórios, projetada para separar o código da aplicação da configuração da infraestrutura:

1.  **Repositório da Aplicação (`pythonCI-CD`):**
    Contém o código-fonte da aplicação FastAPI e o pipeline de CI (Integração Contínua) que testa o código e constrói as imagens Docker.

2.  **Repositório de Manifestos (`manifestsCI-PythonAPI` - este repositório):**
    Contém a configuração de implantação do Kubernetes. É monitorado por nossa ferramenta de CD (Entrega Contínua), o **ArgoCD**. 

Qualquer alteração no estado da aplicação em produção **deve** ser feita através de um commit neste repositório.

## 📁 Conteúdo do Repositório

  * **`deployment.yaml`**: Define o `Deployment` do Kubernetes. Este recurso gerencia os Pods da nossa aplicação, especificando qual imagem Docker deve ser executada, o número de réplicas e a estratégia de atualização. A tag da imagem neste arquivo é atualizada automaticamente pelo nosso pipeline de CI.

  * **`service.yaml`**: Define o `Service` do Kubernetes. Este recurso expõe nossa aplicação como um serviço de rede, permitindo que ela seja acessada por outras aplicações dentro do cluster ou externamente.

## 🔄 O Fluxo de Trabalho GitOps

O processo de implantação é totalmente automatizado e centrado neste repositório:

1.  **Início no Repositório da Aplicação:** Um desenvolvedor finaliza uma nova versão da aplicação e cria uma tag Git (ex: `v0.1.0`) no repositório `pythonCI-CD`.

2.  **Pipeline de CI é Acionado:** A criação da tag aciona um workflow do GitHub Actions que:
    a. Executa os testes automatizados.
    b. Se os testes passarem, constrói uma nova imagem Docker e a publica no Docker Hub com a tag `v0.1.0`.

3.  **Abertura de Pull Request:** O workflow, então, clona este repositório (`manifestsCI-PythonAPI`), atualiza a tag da `image` no arquivo `deployment.yaml` para a nova versão (`v0.1.0`) e abre um **Pull Request** com essa alteração.

4.  **Revisão e Aprovação (Controle Humano):** A equipe de DevOps ou o líder técnico revisa o Pull Request. Este é o ponto de controle de governança, onde a implantação para o ambiente é formalmente aprovada.

5.  **Merge e Sincronização do ArgoCD:**
    a. Uma vez que o Pull Request é mesclado (merged) na branch `main`, o ArgoCD, que está constantemente monitorando este repositório, detecta a alteração.
    b. Como a política de sincronização está configurada para `Automated`, o ArgoCD inicia imediatamente o processo de sincronização. [2]

6.  **Implantação no Kubernetes:** O ArgoCD aplica o novo manifesto ao cluster Kubernetes, que por sua vez realiza um *rolling update*, substituindo os Pods antigos pelos novos com a imagem atualizada, **sem downtime**. [3]

## 🚀 Gerenciamento de Implantações

### Aprovar uma Nova Versão

Para implantar uma nova versão, simplesmente **revise e aprove o Pull Request** gerado automaticamente pelo pipeline de CI. O merge na branch `main` é a ação que efetivamente aprova a implantação.

### Realizar um Rollback

Uma das grandes vantagens do GitOps é a facilidade de rollback. Para reverter para uma versão anterior:

1.  Navegue até o histórico de commits da branch `main` deste repositório.
2.  Use o comando `git revert` para reverter o commit de merge que introduziu a versão com problemas.
3.  Faça o `push` da reversão para a branch `main`.

O ArgoCD detectará que o estado do repositório mudou para uma versão anterior e automaticamente fará o rollback da aplicação no cluster para corresponder ao estado revertido.

## 🔗 Configuração do ArgoCD

O ArgoCD deve ser configurado para monitorar este repositório com as seguintes configurações:

  * **Repository URL:** `https://github.com/raian2209/manifestsCI-PythonAPI.git`
  * **Revision:** `HEAD`
  * **Path:** `.`
  * **Destination Cluster:** `https://kubernetes.default.svc`
  * **Namespace:** `aplication` ( o namespace de destino da aplicação tentando separar serviços pelos namespaces)
  * **Sync Policy:** `Automated`
      * **Prune Resources:** `true` (para remover recursos que não existem mais no Git)
      * **Self Heal:** `true` (para corrigir desvios de configuração manuais no cluster) 