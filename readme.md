# DB Sentinel | MySQL

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Este repositório cobre **MySQL**. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são validados em laboratório antes da publicação.

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [InnoDB Cluster Single-Primary com MySQL Router](InnoDB-Cluster/innodb-cluster-single-primary-router.md) | Cluster de 3 nós com um primário (escrita) e dois secundários (leitura), failover automático e porta única de RW e RO via Router | Validado |
| InnoDB Cluster Multi-Primary | Migração de Single-Primary para Multi-Primary, com escrita em todos os nós | Planejado |
| Recuperação do InnoDB Cluster | Perda de quórum e queda total do cluster | Planejado |

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MySQL Server | 9.7.4 LTS |
| MySQL Shell | 26.7.1 |
| MySQL Router | 26.7.0 |
| Virtualização | Oracle VirtualBox |

Cada procedimento informa as versões exatas que usa.

## Padrão dos documentos

Todo procedimento deste repositório segue o mesmo formato:

- Topologia desenhada em diagrama.
- Versões e pré-requisitos explícitos.
- Passos numerados, com comandos prontos para copiar.
- Resultado esperado após cada etapa.
- Somente o que foi validado.

## Estrutura

```
MySQL/
├── README.md
└── InnoDB-Cluster/
    └── innodb-cluster-single-primary-router.md
```

## Como usar

1. Leia a seção de versões e pré-requisitos do procedimento.
2. Execute os passos na ordem indicada.
3. Valide cada etapa antes de seguir para a próxima.
4. Troque todas as senhas de exemplo antes de usar em qualquer ambiente real.

## Aviso

Os procedimentos foram validados em laboratório. Teste em ambiente de homologação antes de aplicar em produção.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
