# AutoInsight — Sprint 3 de Cybersecurity

Documentação e evidências de Cybersecurity do Challenge Ford FIAP 2026.

## Equipe

| Nome | RM |
|---|---|
| Ali Andrea Mamani Molle | 558052 |
| Guilherme Linard F. R. Gozzi | 555768 |
| Lucas Vasquez Silva | 555159 |

## Repositórios e aplicação

| Recurso | Link |
|---|---|
| Código atualizado da API | [Sprint-Soa-Ford](https://github.com/AliAndrea1/Sprint-Soa-Ford) |
| Aplicativo mobile | [fiap-mdi-sprint-autoinsight](https://github.com/AliAndrea1/fiap-mdi-sprint-autoinsight) |
| Documentação interativa da API | [Swagger UI](https://sprint-soa-ford-production.up.railway.app/swagger-ui.html) |
| Pipeline de segurança | [Execução aprovada no GitHub Actions](https://github.com/AliAndrea1/Sprint-Soa-Ford/actions/runs/36296251088) |

Este repositório reúne a análise e as evidências de Cybersecurity. A implementação atual da API é mantida no repositório **Sprint-Soa-Ford**.

## Escopo e estado da entrega

A API AutoInsight foi desenvolvida em Spring Boot, utiliza MySQL no Railway e é consumida pelo aplicativo Android por HTTPS.

Na [execução do pipeline de 27/09/2026](https://github.com/AliAndrea1/Sprint-Soa-Ford/actions/runs/36296251088), passaram os testes Java, a análise estática com Semgrep e a verificação de segredos com Gitleaks. O deploy do Railway também apresentou sucesso nessa execução. As pendências de segurança indicadas neste documento ainda precisam ser tratadas e comprovadas separadamente.

## 1. Pipeline DevSecOps

O [workflow de segurança](https://github.com/AliAndrea1/Sprint-Soa-Ford/blob/master/.github/workflows/security.yml) está no repositório da API e executa em `push`, `pull_request` e por acionamento manual. A [configuração do Dependabot](https://github.com/AliAndrea1/Sprint-Soa-Ford/blob/master/.github/dependabot.yml) verifica atualizações de dependências Maven e GitHub Actions.

```mermaid
flowchart TD
    A["Push ou pull request na API"] --> B["GitHub Actions"]
    B --> C["Testes Java"]
    B --> D["Semgrep"]
    B --> E["Gitleaks"]
    C --> F["Resultado e revisão"]
    D --> F
    E --> F
    F --> G["Correções e nova execução"]
```

| Verificação | O que faz | Risco abordado | Evidência |
|---|---|---|---|
| Testes Java | Executa testes de autenticação, JWT, serviço de veículos, erros e permissões | Regressões nas regras de negócio e no controle de acesso | [Execução aprovada](https://github.com/AliAndrea1/Sprint-Soa-Ford/actions/runs/36296251088) |
| Semgrep | Analisa estaticamente o código Java | Padrões de implementação inseguros cobertos pelas regras utilizadas | [Job do Semgrep](https://github.com/AliAndrea1/Sprint-Soa-Ford/actions/runs/36296251088/job/108555535038) |
| Gitleaks | Examina o histórico Git em busca de segredos identificáveis | Exposição de credenciais detectáveis pelas regras da ferramenta | [Job do Gitleaks](https://github.com/AliAndrea1/Sprint-Soa-Ford/actions/runs/36296251088/job/108555535181) |
| Dependabot | Propõe atualizações de dependências | Uso de componentes desatualizados | [Configuração do Dependabot](https://github.com/AliAndrea1/Sprint-Soa-Ford/blob/master/.github/dependabot.yml) |

O Dependabot abre propostas de atualização, mas elas devem ser avaliadas antes do merge. Uma atualização de versão principal, como Spring Boot 3 para 4, pode exigir alterações no projeto e novos testes.

### Avaliação dos pull requests do Dependabot

Os PRs criados antes da correção do workflow falhavam com `./mvnw: Permission denied`. Após alterar a execução para `bash ./mvnw`, os PRs foram atualizados com `@dependabot rebase` para usar o pipeline corrigido. Os resultados observados foram:

| PR | Atualização proposta | Resultado observado | Decisão |
|---|---|---|---|
| [#2](https://github.com/AliAndrea1/Sprint-Soa-Ford/pull/2) | Spring Boot 3.5.14 → 4.1.1 | Falha de compilação em `VehicleSecurityTest`: o pacote de `@WebMvcTest` mudou no Spring Boot 4 | Manter 3.5.14; planejar a migração e testar a API e o Swagger |
| [#4](https://github.com/AliAndrea1/Sprint-Soa-Ford/pull/4) | `jjwt-api` 0.12.6 → 0.13.0 | A atualização isolada apresentou seis erros nos testes por incompatibilidade com a implementação antiga | Avaliar `jjwt-api`, `jjwt-impl` e `jjwt-jackson` juntos antes do merge |
| [#6](https://github.com/AliAndrea1/Sprint-Soa-Ford/pull/6) | Springdoc 2.8.8 → 3.1.1 | `VehicleSecurityTest` falhou durante a inicialização do Spring | Manter Springdoc 2.x enquanto a API usar Spring Boot 3 |
| [#8](https://github.com/AliAndrea1/Sprint-Soa-Ford/pull/8) | `jjwt-impl` 0.12.6 → 0.13.0 | Os checks passaram após a atualização do PR | Aguardar avaliação conjunta dos três módulos JJWT antes do merge |

Os demais PRs seguem em análise individual. Um check aprovado confirma apenas as verificações configuradas para aquele PR; não autoriza automaticamente o merge ou o deploy.

**Limite atual:** o sucesso dos jobs não garante, sozinho, que todo o código é seguro. Ainda é necessário revisar achados não detectados pelas ferramentas, proteger a branch e confirmar como o deploy do Railway é liberado. O workflow não impede automaticamente um deploy feito diretamente após um `push`.

### Evidência visual do pipeline

<img width="1843" height="953" alt="pipeline-sucesso" src="https://github.com/user-attachments/assets/7e6ad221-7a1c-4b8e-bb55-47627a409423" />

## 2. Segurança do código e da infraestrutura

| Controle | Onde está implementado | Como verificar |
|---|---|---|
| JWT assinado e com expiração | `JwtUtil.java` e `JwtFilter.java` | `JwtUtilTest` e `JwtFilterTest` |
| Perfis `ADMIN` e `ANALYST` | `SecurityConfig.java` | `VehicleSecurityTest`: acesso sem token, leitura permitida e escrita negada |
| Limite de requisições por IP | `RateLimitFilter.java` | Resposta HTTP 429 ao atingir o limite |
| Histórico criptografado com AES/GCM | `CryptoUtils.java` e `SearchHistoryService.java` | Conferir os dados persistidos sem expor a chave |
| Validação de entrada | DTOs e `VehicleController.java` | Enviar entradas inválidas e verificar HTTP 400 |
| HTTPS externo | Domínio público da API no Railway | Acessar a API por `https://` |

Os arquivos citados estão em [`src/main/java/com/autoinsight/autoinsight_api`](https://github.com/AliAndrea1/Sprint-Soa-Ford/tree/master/src/main/java/com/autoinsight/autoinsight_api). Os testes estão em [`src/test/java`](https://github.com/AliAndrea1/Sprint-Soa-Ford/tree/master/src/test/java).

### Pendências prioritárias

O `AuthController` passou a ler as senhas das variáveis `ADMIN_PASSWORD` e `ANALYST_PASSWORD`. Os valores padrão de `DB_PASSWORD`, `JWT_SECRET` e `CRYPTO_KEY` foram retirados do `application.properties`. As variáveis necessárias foram configuradas no Railway, e os dois perfis tiveram o login testado no aplicativo após o deploy. O `CryptoUtils` passou a interromper a operação quando a criptografia falha; o `CryptoUtilsTest` verifica que não há retorno em texto aberto. A execução local reportou 19 testes aprovados. Consulte [código e testes da API](https://github.com/AliAndrea1/Sprint-Soa-Ford).

A chave JWT foi renovada. A chave AES do histórico permanece compatível com os registros existentes; sua troca exige migração planejada dos dados. Ainda falta verificar backup e restauração e reavaliar a proteção da branch e o armazenamento do token no aplicativo.

Tokens, senhas e chaves não devem aparecer nos prints de evidência.

## 3. Observabilidade e resposta a incidentes

O `AuthController` registra tentativas de login bem-sucedidas e malsucedidas. O `AuditLogFilter` registra usuário, método HTTP, endpoint, IP, status da resposta e horário. Respostas 401 e 403 também recebem destaque nos logs.

Esses registros ajudam na investigação, mas **logs não equivalem a um painel com alertas configurados**. Ainda é necessário comprovar quais métricas e alertas estão disponíveis e ativos na infraestrutura utilizada.

| Sinal a acompanhar | Possível significado | Ação inicial |
|---|---|---|
| Muitas respostas 401 | Tentativas de autenticação inválidas | Verificar origem, volume e horário |
| Muitas respostas 403 | Tentativas de acessar funções sem permissão | Conferir conta e endpoint envolvidos |
| Aumento de respostas 429 | Volume de requisições acima do limite | Investigar origem e impacto |
| Aumento de respostas 5xx | Falha da API ou de uma dependência | Consultar logs da aplicação e do banco |
| API indisponível | Interrupção do serviço | Verificar o deploy e a infraestrutura |

### Evidência dos logs

![Registros de autenticação e auditoria da API no Railway](logs-autenticacao.JPG)

### Dashboard de infraestrutura

O painel Metrics do Railway acompanha o consumo de CPU, memória e rede
do serviço da API. Os logs de autenticação e auditoria complementam esses
gráficos na investigação de incidentes.

![Métricas da API no Railway](dashboard-railway.JPG)

O painel acima apresenta métricas de infraestrutura; taxas de erro e latência da aplicação exigem instrumentação ou análise específica dos logs. A configuração de alertas ainda precisa ser comprovada.

### Fluxo de resposta a incidentes

1. **Detecção:** identificar alerta, erro ou comportamento anormal.
2. **Análise:** reunir horário, endpoints afetados, logs e impacto, sem copiar tokens ou senhas para a documentação.
3. **Contenção:** restringir o acesso afetado ou revogar credenciais comprometidas.
4. **Erradicação:** corrigir a causa no código ou na configuração.
5. **Recuperação:** restaurar o serviço e testar novamente a API e os fluxos do aplicativo.
6. **Registro:** documentar a causa, a solução e as ações para evitar recorrência.

Backup do MySQL e teste de restauração precisam de evidência própria antes de serem apresentados como controles concluídos.

### Procedimento de resposta para a API AutoInsight

| Situação detectada | Verificação inicial | Contenção e recuperação | Registro esperado |
|---|---|---|---|
| Sequência incomum de logins inválidos ou respostas 401/403 | Conferir horário, rota, usuário e IP nos registros de auditoria; comparar com acessos esperados | Restringir credenciais afetadas; trocar a senha configurada no Railway quando necessário; confirmar login dos perfis após a alteração | Horário, evidências sem senhas ou tokens, decisão e resultado do novo teste |
| Aumento de erros 5xx ou API indisponível | Conferir logs do deploy, estado do serviço MySQL e métricas do Railway | Corrigir configuração ou versão afetada; confirmar Swagger, login e consulta de veículos no aplicativo | Causa identificada, intervalo da indisponibilidade e testes após recuperação |
| Suspeita de exposição de chave ou token | Identificar o material exposto e onde foi publicado, sem reproduzir seu valor no relatório | Revogar ou substituir a credencial; gerar nova chave JWT quando necessário. A troca da chave AES requer migração dos históricos criptografados antes da substituição | Local da exposição, credenciais rotacionadas e validação do histórico |

O painel do Railway apresenta métricas e os logs permitem investigação manual; **não há evidência de alertas automáticos configurados**. A equipe deve acompanhar os registros e testar o fluxo de resposta periodicamente.

### Situação do backup e recuperação

Em 27/09/2026, a interface do serviço MySQL no Railway mostrou `No Backups` e informou que a criação de backups e a recuperação em um momento específico (PITR) dependem do plano Pro. O arquivo `backup.zip` gerado pela opção **Backup Connections** do MySQL Workbench contém configurações de conexão, não os dados do banco. A tentativa de exportação pelo Workbench encontrou incompatibilidade entre o `mysqldump` local 8.0.46 e o MySQL 9.4.0 do Railway, portanto não foi usada como prova de backup.

**Rotina proposta:** usar uma ferramenta de exportação compatível com a versão do servidor para gerar uma cópia lógica, guardar o arquivo com acesso restrito fora do repositório, restaurá-lo em um banco separado e verificar as tabelas e uma amostra de registros sem divulgar dados pessoais. Essa rotina e o teste de restauração **ainda não foram executados**; não restaurar diretamente no banco de produção para produzir evidência.

## 4. Análise de riscos e conformidade

### Modelo STRIDE

| Categoria | Cenário no AutoInsight | Controle existente ou ação necessária |
|---|---|---|
| Spoofing | Uso indevido de conta ou token | JWT; senhas por variáveis de ambiente e revisão periódica de acessos |
| Tampering | Alteração indevida de veículo | Escrita restrita a `ADMIN`; testar e auditar operações |
| Repudiation | Usuário negar uma operação realizada | Trilha de auditoria; proteger acesso e retenção dos logs |
| Information disclosure | Exposição de segredos, token ou histórico | Gitleaks, revisão de segredos e armazenamento do token |
| Denial of service | Excesso de requisições | Rate limiting; verificar comportamento atrás do proxy |
| Elevation of privilege | `ANALYST` tentar executar ações de `ADMIN` | RBAC e testes de resposta 403 |

### Referências para revisão

A revisão de segurança considera os seguintes grupos de requisitos:

- **OWASP ASVS:** autenticação, gerenciamento de sessão, autorização, validação e registros de segurança.
- **OWASP API Security Top 10:** autorização por objeto e por função, consumo de recursos e configuração da API.
- **OWASP Mobile Top 10:** armazenamento local do token, comunicação com a API e segurança do aplicativo Android.

Essa relação orienta a revisão. **Ela não representa uma certificação ou conformidade integral com o ASVS.**

### Privacidade e LGPD

O projeto deve identificar quais dados pessoais aparecem nas contas, nos endereços IP e nos logs. Também precisa definir finalidade do tratamento, acesso mínimo, retenção e procedimento de exclusão quando aplicável.

O histórico de buscas pode revelar atividades profissionais dos usuários. O acesso ao banco e às evidências deve ser limitado. Telemetria e localização não foram confirmadas como dados coletados pelo aplicativo; devem ser analisadas caso sejam adicionadas no futuro.

### Plano de segurança contínua

| Rotina | Frequência proposta | Procedimento e evidência | Estado |
|---|---|---|---|
| Revisão de dependências | Semanal e antes de cada entrega | Revisar PRs do Dependabot, checar compatibilidade e executar testes antes do merge; registrar a decisão em cada PR | Em execução: PRs #2, #4, #6 e #8 analisados |
| Testes de segurança | A cada push e pull request da API | Executar testes de autenticação, JWT, autorização e criptografia; acompanhar Semgrep e Gitleaks no Actions | Implementado no pipeline; conferir a execução do commit entregue |
| Auditoria de permissões | Mensal e após alteração nos endpoints | Revisar a matriz `ADMIN`/`ANALYST`; testar 401 sem token, 403 sem permissão e 200 com acesso autorizado | Testes de veículos realizados; demais rotas exigem revisão |
| Análise de logs e incidentes | Semanal e após comportamento suspeito | Examinar tentativas de login, 401, 403, 429 e 5xx; registrar causa e ação tomada sem copiar dados sensíveis | Logs e métricas capturados; alertas ainda pendentes |
| Backup e recuperação do MySQL | Frequência a definir conforme recursos do banco e necessidade de retenção | Fazer exportação compatível com MySQL 9.4, guardar com acesso restrito e restaurar em ambiente separado | Pendente: backups nativos do Railway exigem plano Pro; exportação e restauração ainda não comprovadas |

### Checklist de conformidade da entrega

| Verificação | Situação | Evidência ou pendência |
|---|---|---|
| Autenticação, JWT e expiração | Implementado | `JwtUtilTest`, `JwtFilterTest` e testes de login |
| Autorização por perfil | Parcialmente verificado | `VehicleSecurityTest`; ampliar os testes para histórico e auditoria |
| Proteção contra excesso de requisições | Implementado, validação pendente | `RateLimitFilter`; testar HTTP 429 e conferir IP real atrás do proxy Railway |
| Criptografia do histórico | Implementado | `CryptoUtilsTest` e código da API; rotação da chave AES ainda exige migração dos dados |
| Análise de código e segredos no CI | Implementado | Jobs Semgrep e Gitleaks; revisar resultados em cada execução |
| Revisão OWASP ASVS/API/Mobile | Mapeamento inicial | Revisão detalhada por requisito ainda pendente; não declarar certificação |
| Dados pessoais e LGPD | Identificação inicial | Definir finalidade, retenção, acesso e exclusão para contas, IPs, logs e histórico |
| Alertas de aplicação | Pendente | Métricas de infraestrutura e logs não comprovam alertas ativos |
| Backup e restauração | Pendente | Railway sem backups no plano atual; executar exportação compatível e testar restauração em banco separado |

## Evidências e próximos passos

| Evidência ou ação | Situação |
|---|---|
| Execução de testes Java, Semgrep e Gitleaks | [Aprovada no GitHub Actions](https://github.com/AliAndrea1/Sprint-Soa-Ford/actions/runs/36296251088) |
| Print dos checks aprovados | Incluído na seção Pipeline DevSecOps |
| Avaliação dos PRs do Dependabot | PRs #2, #4, #6 e #8 analisados acima; demais PRs ainda requerem revisão antes de qualquer merge |
| Correção das credenciais e valores padrão | Implementada na API; validar pelo commit e nova execução do pipeline |
| Métricas de infraestrutura e logs | Prints incluídos nas seções de observabilidade; alertas ainda pendentes |
| Backup e restauração testada | Pendente: serviço Railway mostra `No Backups`; rotina alternativa está descrita na seção 3 |
| Plano de segurança contínua e checklist | Incluídos na seção 4; itens parciais e pendentes identificados |
