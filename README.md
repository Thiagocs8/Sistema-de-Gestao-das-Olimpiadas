# Sistema de Gestão das Olimpíadas (SGO)

Trabalho de modelagem UML para o **Sistema de Gestão das Olimpíadas (SGO)**, desenvolvido como parte da disciplina de Projeto de Software.

O sistema tem como objetivo coordenar os diferentes aspectos de um evento olímpico: cadastro e alocação de competições, inscrição de atletas, registro de resultados e geração de relatórios de medalhas por país. A solução foi modelada seguindo uma arquitetura de **microsserviços em nuvem (AWS)**, com processamento assíncrono via filas SQS para operações pesadas (inscrições em massa, registro de resultados e geração de relatórios).

---

## 📑 Sumário

- [Descrição do Sistema](#-descrição-do-sistema)
- [Regras de Negócio](#-regras-de-negócio)
- [Histórias de Usuário](#-histórias-de-usuário)
- [Arquitetura](#-arquitetura)
- [Diagramas UML](#-diagramas-uml)
  - [Diagrama de Caso de Uso](#diagrama-de-caso-de-uso)
  - [Diagrama de Classes e Pacotes](#diagrama-de-classes-e-pacotes)
  - [Diagrama de Componentes](#diagrama-de-componentes)
  - [Diagrama de Implantação](#diagrama-de-implantação)
- [Estrutura do Repositório](#-estrutura-do-repositório)
- [Tecnologias Utilizadas](#-tecnologias-utilizadas)
- [Como Regenerar os Diagramas](#-como-regenerar-os-diagramas)

---

## 📋 Descrição do Sistema

Com a chegada das Olimpíadas, é necessário um sistema robusto para coordenar competições, inscrições de atletas, alocação de locais e controle de resultados em escala global. O SGO atende três perfis de usuário — **Administrador**, **Atleta** e **Juiz** — oferecendo funcionalidades específicas para cada papel, garantindo isolamento de domínios e alta disponibilidade através de uma arquitetura distribuída.

---

## 📐 Regras de Negócio

| # | Regra |
|---|---|
| RN01 | O sistema deve permitir o cadastro de competições com modalidade, data, horário, local e lista de atletas inscritos. |
| RN02 | Cada atleta pode se inscrever em várias competições, mas representa apenas um país. |
| RN03 | Um local não pode abrigar mais de uma competição no mesmo horário (verificação obrigatória de conflito). |
| RN04 | Após cada competição, devem ser registrados os atletas vencedores em 1º, 2º e 3º lugares (ouro, prata e bronze). |
| RN05 | O sistema deve gerar relatórios de medalhas por país, consolidando ouro, prata e bronze. |

---

## 👥 Histórias de Usuário

### US01 — Cadastrar Competição
> **Como** Administrador, **quero** cadastrar uma competição informando modalidade, data, horário e local, **para que** os atletas possam se inscrever e o evento seja oficialmente registrado no sistema.
>
> **Critérios de aceitação:**
> - Não deve permitir cadastro sem modalidade, data ou horário definidos.
> - O local informado deve estar disponível na data/horário escolhidos.
> - A confirmação deve ser persistida no banco de competições (S1).

### US02 — Inscrever Atleta
> **Como** Atleta, **quero** me inscrever em uma competição da minha modalidade, **para que** eu possa participar dos jogos representando meu país.
>
> **Critérios de aceitação:**
> - O atleta deve estar vinculado a um país antes de se inscrever.
> - Não deve permitir inscrição em competições já encerradas.
> - A inscrição é processada de forma assíncrona via fila SQS, permitindo picos de demanda.

### US03 — Alocar Local
> **Como** Administrador, **quero** alocar um local físico para uma competição, **para que** ela tenha um espaço definido para sua realização.
>
> **Critérios de aceitação:**
> - O sistema deve verificar conflitos de horário automaticamente.
> - Não deve permitir alocação se já existir outra competição no mesmo local e horário.

### US04 — Verificar Conflito de Horário
> **Como** Administrador, **quero** que o sistema verifique conflitos de horário ao alocar locais, **para que** seja garantido que um local não receberá duas competições simultâneas.
>
> **Critérios de aceitação:**
> - A verificação deve ocorrer antes da confirmação da alocação.
> - Em caso de conflito, o sistema deve recusar a operação e exibir mensagem clara.

### US05 — Registrar Resultados
> **Como** Juiz, **quero** registrar os resultados de uma competição (medalhas de ouro, prata e bronze), **para que** os vencedores sejam definidos oficialmente.
>
> **Critérios de aceitação:**
> - Cada competição deve ter exatamente três atletas medalhistas.
> - Os atletas medalhistas devem estar previamente inscritos na competição.
> - O registro é enfileirado para processamento assíncrono e atualização do banco de resultados (S4).

### US06 — Gerar Relatório de Medalhas
> **Como** Administrador, **quero** gerar um relatório de medalhas por país, **para que** eu possa acompanhar o desempenho de cada nação participante.
>
> **Critérios de aceitação:**
> - O relatório deve listar ouro, prata, bronze e total de medalhas por país.
> - Deve permitir ordenação pelo total de medalhas conquistadas.
> - A geração é assíncrona (fila + worker dedicado), liberando o usuário imediatamente.

### US07 — Consultar Competições
> **Como** Atleta ou Administrador, **quero** consultar a lista de competições, **para que** eu possa visualizar modalidades, datas, locais e atletas inscritos.
>
> **Critérios de aceitação:**
> - A consulta deve estar disponível para Administradores e Atletas autenticados.
> - Deve permitir filtros por modalidade e data.

---

## 🏗 Arquitetura

O sistema segue uma arquitetura de **microsserviços em nuvem AWS**, com os seguintes princípios:

- **Separação por domínio:** cada microsserviço encapsula um domínio do negócio (Competições, Atletas/Inscrições, Locais/Alocação, Resultados, Relatórios, Segurança).
- **Banco de dados por serviço:** cada microsserviço possui seu próprio banco isolado (S1 a S6), garantindo independência de evolução e deploy.
- **Processamento assíncrono:** operações pesadas (inscrições em massa, registro de resultados, geração de relatórios) usam filas **AWS SQS** consumidas por *workers* dedicados.
- **API Gateway centralizado:** ponto único de entrada para o front-end, responsável por roteamento e segurança.
- **Front-end desacoplado:** SPA hospedada em servidor web, consumindo a API via HTTPS/REST.

### Mapa dos serviços e bancos

| Microsserviço | Banco AWS | Responsabilidade |
|---|---|---|
| Serviço de Competições | S1 — BD Competições | Cadastro e consulta de competições |
| Serviço de Atletas/Inscrições | S2 — BD Atletas | Cadastro de atletas e inscrições em competições |
| Serviço de Locais/Alocação | S3 — BD Locais | Cadastro de locais e verificação de conflitos |
| Serviço de Resultados | S4 — BD Resultados | Registro e consulta de medalhas por competição |
| Serviço de Segurança | S5 — BD Segurança | Autenticação e autorização |
| Serviço de Relatórios | S6 — BD Relatórios | Geração e cache de relatórios consolidados |

---

## 📊 Diagramas UML

### Diagrama de Caso de Uso

Apresenta os atores (Administrador, Atleta, Juiz) e as principais funcionalidades do sistema. O caso de uso *Alocar Local* inclui obrigatoriamente *Verificar Conflito de Horário* via relacionamento `<<include>>`.

<(https://github.com/Thiagocs8/Sistema-de-Gestao-das-Olimpiadas/blob/main/imagens/Diagrama%20de%20Casos%20de%20Uso.png)>

---

### Diagrama de Classes e Pacotes

Estrutura de classes do domínio organizada em pacotes seguindo o padrão **MVC + Repository**:

- **`controller`** — Camada de entrada da API, recebe requisições e delega para os serviços (`CompeticaoController`, `AtletaController`, `LocalController`, `ResultadoController`, `RelatorioController`).
- **`service`** — Camada de regras de negócio (`CompeticaoService`, `AtletaService`, `LocalService`, `ResultadoService`, `RelatorioService`).
- **`repository`** — Camada de persistência (`CompeticaoRepository`, `AtletaRepository`, `LocalRepository`, `ResultadoRepository`, `PaisRepository`).
- **`model`** — Entidades de domínio: **Competicao**, **Atleta**, **Local**, **Resultado** e **Pais**, com suas associações e multiplicidades. A classe *Resultado* mantém referências separadas para os atletas medalhistas (ouro, prata e bronze).

<img src="imagens/diagrama-de-classes.png" alt="Diagrama de Classes e Pacotes" width="900"/>

---

### Diagrama de Componentes

Visão lógica dos componentes do sistema. O *API Gateway* expõe a API SGO ao front-end e roteia as requisições REST para os microsserviços. Operações assíncronas trafegam por filas SQS e são processadas pelos workers correspondentes.

<img src="imagens/diagrama-de-componentes.png" alt="Diagrama de Componentes" width="900"/>

---

### Diagrama de Implantação

Arquitetura física do sistema na nuvem AWS: Dispositivo do Usuário → Servidor Web (Front-end) → API Gateway → 6 microsserviços, com filas SQS, workers e bancos dedicados por domínio. Protocolos identificados: HTTPS no acesso externo, REST entre Gateway e serviços, HTTP entre serviços e filas, JDBC/SQL para os bancos.

<img src="imagens/diagrama-de-implantacao.png" alt="Diagrama de Implantação" width="900"/>

---

## 📁 Estrutura do Repositório

```
sistema-gestao-olimpiadas/
├── README.md
├── imagens/
│   ├── diagrama-de-caso-de-uso.png
│   ├── diagrama-de-classes.png
│   ├── diagrama-de-pacotes.png
│   ├── diagrama-de-componentes.png
│   └── diagrama-de-implantacao.png
└── codigos/
    ├── diagrama-de-caso-de-uso.puml
    ├── diagrama-de-classes.puml
    ├── diagrama-de-pacotes.puml
    ├── diagrama-de-componentes.puml
    └── diagrama-de-implantacao.puml
```

> **Observação:** os arquivos `diagrama-de-classes.*` e `diagrama-de-pacotes.*` contêm o mesmo diagrama integrado, já que as classes do sistema estão organizadas dentro dos pacotes. Ambos são listados separadamente para atender ao checklist de entregáveis do enunciado.

---

## 🛠 Tecnologias Utilizadas

### Modelagem
- **[PlantUML](https://plantuml.com/)** — Modelagem textual dos diagramas UML.
- **[Guia PlantUML](https://plantuml.com/guide)** — Referência de sintaxe.
- **[PlantUML API (Python)](https://github.com/joaopauloaramuni/projeto-de-software/tree/main/PROJETOS/Python/Projeto%20PlantUML%20API)** — Automação na geração das imagens.

### Arquitetura referenciada nos diagramas
- **AWS** — Plataforma de nuvem (SQS para filas, EC2/ECS para microsserviços, RDS para bancos).
- **API Gateway** — Ponto único de entrada das requisições REST.
- **Microsserviços** — Separação por domínio com banco isolado por serviço.

---

## 🔄 Como Regenerar os Diagramas

Com o PlantUML instalado localmente:

```bash
# Gera todas as imagens PNG na pasta imagens/
plantuml -tpng -o ../imagens codigos/*.puml
```

Para usar o servidor público sem instalação:

```bash
# Cada arquivo .puml pode ser renderizado online em:
# http://www.plantuml.com/plantuml/uml/
```

---

## 👤 Autor

Vinicius Oliveira Ramos e Thiago Soares Costa — Trabalho desenvolvido para a disciplina de Projeto de Software.
