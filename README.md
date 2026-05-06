📑 GAD - Global Architecture Document: Sistema de Seguros Omni (Enterprise Version)
Versão: 2.0 (Full Data Governance)

Data: 06/05/2026

Status: Baseline para Desenvolvimento
1. Visão Geral e Estilo Arquitetural
O sistema utiliza uma arquitetura de Microserviços Híbridos com Persistência Poliglota. O foco é o desacoplamento total entre o motor contratual (SQL Server) e as especialidades de produto (NoSQL).

		1.1 Diagrama de Macro-Arquitetura
graph TD
    User((Usuário/Blazor)) --> GW[API Gateway]
    subgraph "Camada de Microserviços (Domain Driven)"
        GW --> IdS[Identity Service]
        GW --> Core[Core Insurance Service]
        GW --> Auto[Auto Specialist Service]
    end

    subgraph "Persistência Poliglota"
        IdS --> DB1[(Auth SQL DB)]
        Core --> DB2[(Core SQL DB)]
        Auto --> DB3[(Auto NoSQL)]
    end

    Core -.-> |Event: ApoliceEmitida| Bus{Message Bus}
    Bus -.-> |Queue| Audit[Audit & Governance Service]
		
2. Governança e Arquitetura de DadosA governança é implementada via Canonical Data Model (CDM), garantindo que o dado tenha o mesmo significado em todos os serviços.2.1 Matriz de Classificação e TaxonomiaTodos os dados devem seguir esta classificação de segurança e sensibilidade:

Classificação	Sigilo	Personal Data (LGPD)	Sensível	Criticidade
Público	Livre	Não	Não	Não Crítico
Interno	Baixo	Não	Não	Crítico
Confidencial	Alto	Sim	Não	Dado Chave
Secreto	Máximo	Sim	Sim	Dado Chave
<img width="595" height="206" alt="image" src="https://github.com/user-attachments/assets/b5c948ef-b383-4960-9e69-2263a38b38ce" />

		2.2 Dicionário de Dados Canônico (Amostra)
Termo de Negócio	Atributo Canônico	Origem	Destino	Classificação
CPF/CNPJ	Customer.TaxId	API Input	SQL Server	Confidencial (Chave)
Prêmio Total	Policy.PremiumAmount	Auto Service	SQL Server	Interno (Crítico)
Perfil de Risco	Risk.Profile	Auto Service	NoSQL	Confidencial
Hash de Senha	Identity.Secret	Identity	SQL (Auth)	Secreto
<img width="774" height="260" alt="image" src="https://github.com/user-attachments/assets/94eb15c0-7a30-41be-8674-ca790b504523" />

3. Diagramas Mermaid (Estrutura Técnica)
		3.1 Modelo Entidade-Relacionamento (MER) Integrado
   
erDiagram
    %% --- BANCO DE IDENTIDADE ---
    AUTH_USER ||--|| CLIENTE : "link_usuario_id"
    
    %% --- BANCO CORE (SQL) ---
    CLIENTE ||--o{ APOLICE : "possui"
    PRODUTO ||--o{ APOLICE : "baseia"
    APOLICE ||--o{ COBERTURA_CONTRATADA : "detalha"
    APOLICE ||--o{ PARCELA : "gera"

    APOLICE {
        Guid Id PK
        Guid ItemReferenciaId "FK Lógica p/ NoSQL"
        String TipoObjeto "Canonical: Object.Type"
        Decimal ValorPremio "Canonical: Policy.Premium"
    }

    %% --- BANCO NO-SQL (DOCUMENTAL) ---
    APOLICE ||--|| ITEM_AUTO_NOSQL : "detalhes_tecnicos"
		
4. Fluxo Sistêmico e Linhagem de Dados (Data Lineage)
		4.1 Ciclo de Vida do Dado (Input-to-Audit)
Ingress (Entrada): O dado entra via API (.NET) e passa pelo Validador de Domínio (ex: Validação de CPF via Regex/Algoritmo).

Canonical Mapping: O dado bruto é mapeado para o Objeto Canônico.

Persistência Híbrida:

Dados Transacionais/Chave -> Gravados no SQL Server.

Dados Flexíveis/Risco -> Gravados no NoSQL.

Egress (Saída): O API Gateway consome ambos os bancos e compõe a resposta para o usuário.

Auditoria: Qualquer alteração em um dado Crítico dispara um evento assíncrono para o log imutável de auditoria.

4.2 Fluxo de Auditoria e Validação
sequenceDiagram
    participant API as API Core
    participant VAL as Validator (Fluent)
    participant DB as Databases (SQL/NoSQL)
    participant LOG as Audit Service (Immutable)

    API->>VAL: Envia Dados (Canonical Model)
    VAL-->>API: OK (Validação de Regras)
    API->>DB: Persiste (SQL e NoSQL)
    API->>LOG: Log Event (Quem, Quando, O que, De/Para)

5. Mecanismos de Controle e Auditoria
Imutabilidade: Os registros de auditoria serão gravados em uma tabela de "Apenas Inserção" (Append-only).

Data Masking: Dados classificados como Confidenciais (como o CPF) devem ser mascarados em logs de depuração (ex: ***.456.***-00).

Snapshots: Para alterações em valores financeiros (Prêmio/Indenização), o sistema deve salvar o JSON do estado anterior e o estado atual.

6. Stack Tecnológica Confirmada
Backend: .NET 8/9 com C#.

Arquitetura: Clean Architecture com Microservices.

Mensageria: RabbitMQ (Comunicação Assíncrona para Auditoria).

Validadores: FluentValidation integrado ao Pipeline do ASP.NET.

Documentação Dinâmica: Swagger/OpenAPI mapeado com os termos do Dicionário de Dados.
