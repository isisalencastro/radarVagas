# Radar de Vagas Tech

Workflow n8n para buscar vagas de tecnologia, normalizar diferentes fontes, reduzir chamadas desnecessárias para IA e salvar no Notion tanto vagas aprovadas quanto descartes auditáveis.

## Arquitetura do workflow

O fluxo principal segue etapas com responsabilidades separadas:

1. **Busca**: APIs RemoteOK, Remotive, Arbeitnow e scrapers Apify para LinkedIn/Indeed.
2. **Normalização**: converte formatos diferentes para um contrato único de vaga.
3. **Deduplicação**: remove duplicadas por ID externo, URL, empresa+título e hash da descrição.
4. **Pré-filtragem**: aplica regras baratas por senioridade, localização, área e tecnologias.
5. **Score inicial**: calcula `score_heuristico` antes da OpenAI.
6. **IA**: avalia apenas vagas com `enviar_para_ia = true`.
7. **Persistência**: salva vagas aprovadas e descartadas no Notion com motivo.
8. **Métricas/logs**: registra volume normalizado, volume enviado para IA e descartes antes da IA.

## Contrato normalizado da vaga

Cada fonte é convertida para estes campos principais:

- `id` / `id_externo`
- `titulo`
- `empresa`
- `descricao`
- `url` / `link`
- `fonte`
- `local`
- `remoto`
- `senioridade`
- `tecnologias`
- `salario`
- `tipo_contrato`
- `idioma`
- `data_publicacao`
- `hash_descricao`

## Score heurístico antes da IA

O node **Normalizar + Remover Duplicadas (Agente 2)** calcula um score barato e configurável. Pontos positivos incluem estágio, júnior, intern, remoto, Brasil, Porto Alegre e tecnologias do perfil como JavaScript, Python, React, Node, PostgreSQL, N8N e OpenAI. Penalidades fortes removem Senior, Staff, Principal, Lead e Manager.

O limite padrão para chamar IA é `45`. Para alterar sem editar o workflow, configure a variável de ambiente:

```bash
RADAR_VAGAS_SCORE_MINIMO_IA=45
```

## Consultas de busca

LinkedIn e Indeed usam uma lista ampliada de consultas em português e inglês, incluindo variações de estágio, intern, júnior, backend, frontend, full stack, JavaScript, Node.js, React, Python, IA e automação. A lista fica no `customBody` dos nodes Apify para facilitar manutenção.

## Persistência e descartes

Nenhuma vaga deve ser descartada silenciosamente. Vagas barradas antes da IA são salvas no Notion com:

- `Status = Descartado`
- `Score heurístico`
- `Motivo descarte`
- `Fonte`, `Empresa`, `Local`, `Senioridade` e tecnologias extraídas

Vagas avaliadas pela IA e rejeitadas também são salvas como descartadas, preservando o motivo retornado pela IA.

## Campos esperados da IA

O prompt exige JSON válido com:

- `compatibilidade`
- `vale_a_pena`
- `prioridade`
- `chance_entrevista`
- `motivo_score`
- `gaps`
- `tecnologias_encontradas`
- `tecnologias_faltantes`
- `senioridade_detectada`
- `empresa_resumo`
- `empresa_confiavel`
- `ghost_job_probabilidade`
- `ingles_obrigatorio`
- `anos_experiencia`
- `curriculo_precisa_personalizacao`
- `carta_apresentacao_recomendada`

## Como adicionar novas fontes

Para incluir uma nova fonte, conecte o node de busca ao merge existente e ajuste apenas a função `normalize(job, source)` caso a fonte use nomes de campos ainda não mapeados. O restante do pipeline reaproveita deduplicação, score, filtros, IA e persistência.
