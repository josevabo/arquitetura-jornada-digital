# Projeto de arquitetura do curso de Jornada Digital ministrado pela ADA

## Contexto
Este esquema trata de um conjunto de aplicações voltado para manter o cadastro de famílias beneficiárias de programas sociais ministrados pelo Governo Federal. Vamos chamá-lo de "Cadastro de Famílias".

A responsabilidade principal destas aplicações é manter a base de dados de famílias beneficiárias de programas sociais voltados aos brasileiros de baixa renda. 

É necessário manter um canal de fácil acesso ao cidadão, para que possa verificar seus dados e mantê-los atualizados.

Entre os dados de cadastro relevantes para o Governo Federal, estão dados de composição familiar (membros, parentesco entre eles), domicílio (endereço, características de saneamento, estrutura e conforto do domicílio), escolaridade, renda e exposição a situações de riscos devidos à classe social e moradia.

## Esquema visual

![system-schema](https://github.com/josevabo/arquitetura-jornada-digital/assets/43576654/f872c613-aeb0-4b7c-8886-55a1f3da308a)

## Elementos 
### Módulos do sistema
Módulos no domínio do conjunto de aplicações do Cadastro de Famílias.

- Frontend: Aplicação web, em React. desenvolvido de forma responsiva para atender usuários Desktop e mobile.
- Backend: Desenvolvido em SpringBoot, realiza a integração com APIs de terceiros e interface com a base de dados da aplicação.
- Serviço background alteração Família: Microsserviço que escuta duas filas para dar prosseguimento aos processos de alteração de dados de família.
- JOB Gerador de relatórios Diários: Aplicação com start schedulado para gerar relatórios em períodos pré definidos.

### Sistemas de terceiros
Projetos mantidos por outros times, que produzem ou consomem dados utilizados por nossos módulos.
- API Cadastro Pessoas (consultas): mantém os dados pessoais de todos os cidadãos Brasileiros 
- API Dados Geográficos: concentra informações de localização utilizadas no cadastro das famílias
- Serviço de cadastro PEssoas (atualizações): realiza atualizações de dados de pessoas
- Sistema de notificações PUSH, e-mail, SMS: gerencia o envio de e-mails, SMS e outras notificações para as pessoas cadastradas
- Serviços de BI: organizam e estruturam dados recebidos a partir de eventos em visões de interesse para diversas áreas de negócio
- Serviços de Pagamentos de Benefícios: a partir de dados atualizados de famílias beneficiárias, é capaz de realizar pagamento de benefícios sociais
- Serviço validador Documentos antifraude: recebe documentos digitalizados, junto de metadados, sendo capaz de responder se o documento é válido

### Elementos de infraestrutura
- RabbitMQ: gerenciador de filas
- Kafka: Streaming de eventos
- Redis: Servidor de Cache compartilhado
- Load Balancer: distribui o acesso às instâncias de aplicações backend
- API Gateway: gerencia o acesso a APIs registradas
- Storage: armazenamento de arquivos como um AWS S3
- B2B: realiza a transferência e recepção de arquivos entre empresas parceiras em ambiente seguro sem exposição à internet
- Scheduler: responsável por monitorar e iniciar jobs periódicos

   
## Fluxos 
Abaixo são descritos os fluxos apresentados no diagrama exibido em imagem acima.

### Fluxo A: Usuário consulta/altera dados família
Processo mais comum, executado a partir do momento que um usuário na internet acessa a aplicaç	ao web para consultar ou alterar dados de sua família.

Seu objetivo é prover ao usuário uma experiência fluida de consulta e alteração de seus dados.

Processo identificado pelas etapas de A.1 a A.6 no diagrama.

- A.1: usuário busca/altera dados da família através do site 
- A.2: frontend repassa requisição ao backend 
- A.3: busca dos dados 
    - 1: busca em cache dados que permitem ser cacheados
    - 2: dados sensíveis sempre são buscados do banco de dados (internos) ou buscados de APIs (externos) 
- A3: alteração dados
    - A.3.2: registra dados alterados pelo usuário, com status pendente de validação 
- A.4: Backend busca detalhes de pessoa e dados de localização em APIs 
    de serviços dedicados a isso.
- A.5: em casos de alteração de dados, arquivos de documentos 
    fornecidos pelo usuário são salvos em storage interno
- A.6: Em caso de alteração de dados, uma mensagem é registrado na fila familias_alterar
    para que o Background JOB possa tratá-la quando disponível (fluxo B)

### Fluxo B: Serviço alteração Família
Processo  assíncrono, iniciado a partir do momento que uma alteração de dados foi solicitada por um usuário e está aguardando em sua respectiva fila de mensagens.

Seu objetivo é garantir a alteração de dados de família de forma assíncrona, performática, sem afetar a disponibilidade da aplicação web.

Processo identificado pelas etapas de B.1 a B.6 no diagrama.

- Background Job escuta as filas de entrada para ele 
    "familias_alterar"  e "documentos_retorno"
- B.1: uma mensagem da fila "familias_alterar" é consumida, contendo dados de alteração de uma família, 
    como renda, membros excluídos, incluídos e alterados, dados de moradia e educação.
- B.2: Acessa documentos eventualmente salvos no storage 
- B.3: Processa as alterações e repassa ao serviço validador de documentos, na fila "documentos_validar", 
    mensagens contendo documentos que dependem de validação antifraude
- B.4: obtém retorno do serviço de validação de documentos consumindo mensagem da fila "documentos_retorno"
- B.5: processa os dados recebidos da validação de documentos e registra no banco de dados se a alteração 
    da família foi acatada
- B.6: distribui mensagens para tópicos de "alteracao_familia" e "alteracao_pessoa", destes tópicos, serviços 
    de notificação e serviços de cadastro de pessoas consumirão 

Obs.: para cada fila no RabbitMQ há uma DLQ que pode ser varrida periodicamente para checar se houve alguma alteraçao de família não finalizada. 


### Fluxo R: Geração de relatórios
Processo  iniciado a partir de um horário prédefinido, através de um scheduler.

Seu objetivo é gerar relatórios diários de alterações, inclusões e outras informações que interessem a áreas de negócio, governo e órgãos externos.

Processo identificado pelas etapas de R.1 a R.4 no diagrama.

- R.1: Um agente scheduler comanda o início da execução de cada JOB de relatórios diários em um horário pré 
    determinado. Por exemplo, todos os dias, às 00h01.
- R.2: O serviço consome os tópicos de alteração de famílias e pessoas no kafka, buscando todas as alterações
    no período para gerar seus relatórios consolidados
- R.3: após processamento, os relatórios são salvos no storage compartilhado com a aplicação
Obs.: os relatórios no storage poderão ser acessados por outras aplicações e áreas de negócio da Caixa. Bem como 
    poderão  ser transferidos para órgãos externos
- R.4: Os relatórios são enviados para órgãos externos atrvés da ferramenta de transferência de arquivos entre 
    empresas, chamada B2B
